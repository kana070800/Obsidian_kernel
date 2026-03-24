
work_struct 주소와 함께 schedule_work()를 호출한다
 schedule_work()
	queue_work_on()
		`__queue_work()`
			insert_work()

schedule_work() 구현부
%%include/linux/workqueue.h%%
```
static inline bool schedule_work(struct work_struct *work)
{
	return queue_work(system_wq, work);
}
```
schedule_work를 통해 큐잉되는 work는 시스템 워크로 큐잉된다

queue_work() 구현부
%%include/linux/workqueue.h%%
```
static inline bool queue_work(struct workqueue_struct *wq,
			      struct work_struct *work)
{
	return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}
```
WORK_CPU_UNBOUND를 전달

queue_work_on() 구현부
%%kernel/workqueue.c%%
```
bool queue_work_on(int cpu, struct workqueue_struct *wq,
		   struct work_struct *work)
{
	bool ret = false;
	unsigned long flags;

	local_irq_save(flags);

	if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))) {
		__queue_work(cpu, wq, work);
		ret = true;
	}

	local_irq_restore(flags);
	return ret;
}
```
local_irq_save(flags), local_irq_restore(flags) 를 통해 인터럽트 잠시 비활성화
work_data_bits(work)으로 work_struct의 data를 읽어옴
	data가 WORK_STRUCT_PENDING_BIT(1) 이면 if 건너뜀 
		test_and_set_bit(A,B)은 리눅스 제공 비트단위 and 연산함수
		결과와 무관하게 B에 A세팅 
	만일  WORK_STRUCT_PENDING_BIT(1)이었다면 이미 큐잉되었다는 뜻


`__queue_work()` 구현부
%%kernel/workqueue.c%%
```
static void __queue_work(int cpu, struct workqueue_struct *wq,
			 struct work_struct *work)
{
	struct pool_workqueue *pwq;
	struct worker_pool *last_pool;
	struct list_head *worklist;
	unsigned int work_flags;
	unsigned int req_cpu = cpu;
...
retry:
	/* pwq which will be used unless @work is executing elsewhere */
	if (wq->flags & WQ_UNBOUND) {
		if (req_cpu == WORK_CPU_UNBOUND)
			cpu = wq_select_unbound_cpu(raw_smp_processor_id());
		pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
	} else {
		if (req_cpu == WORK_CPU_UNBOUND)
			cpu = raw_smp_processor_id();
		pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
	}
	
	last_pool = get_work_pool(work);
	if (last_pool && last_pool != pwq->pool) {
		struct worker *worker;

		spin_lock(&last_pool->lock);

		worker = find_worker_executing_work(last_pool, work);
		
		if (worker && worker->current_pwq->wq == wq) {
			pwq = worker->current_pwq;
		} else {
			/* meh... not running there, queue here */
			spin_unlock(&last_pool->lock);
			spin_lock(&pwq->pool->lock);
		}
	} else {
		spin_lock(&pwq->pool->lock);
	}
...
	/* pwq determined, queue */
	trace_workqueue_queue_work(req_cpu, pwq, work);
...
	if (likely(pwq->nr_active < pwq->max_active)) {
		trace_workqueue_activate_work(work);
		pwq->nr_active++;
		worklist = &pwq->pool->worklist;
		if (list_empty(worklist))
			pwq->pool->watchdog_ts = jiffies;
	} else {
		work_flags |= WORK_STRUCT_DELAYED;
		worklist = &pwq->delayed_works;
	}

	insert_work(pwq, work, worklist, work_flags);

	spin_unlock(&pwq->pool->lock);
}
```
입력인자
	int cpu : WORK_CPU_UNBOUND
	`struct workqueue_struct *wq` : system_wq
	`struct work_struct *work` : work_struct의 주소

흐름
	1단계 : 풀워크 큐 가져오기
		retry레이블에서 raw_smp_processor_id() 로 cpu번호를 얻어냄
		wq_select_unbound_cpu()로 비트마스크와의 and연산까지 수행
	2단계 : 워커 구조체 가져오기
		일반적으로 per_cpu_ptr을 수행하여 workqueue_struct 구조체 cpu_pwqs(percpu)에 접근하여 cpu 주소를 읽어온다
		`__per_cpu_offset[n]`배열에 접근하여 offset을 알아내 주소를 연산
		last_pool = get_work_pool(work); 에서
			work_struct를 통해 get_work_pool을 불러와 worker_pool 구조체의 주소를 last_pool에 저장
			이 때 자료구조 흐름은 [[워크큐 자료구조 overall]] 에 명시됨

2단계 이름을 바꿔야 할까 고민...



	3단계 : ftrace 로그 출력
	4단계 : 워커 풀에 워크의 연결리스트를 등록하고 워커스레드를 깨우기