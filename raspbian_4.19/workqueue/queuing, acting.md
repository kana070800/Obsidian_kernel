
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
	2단계 : 풀 워크큐 주소 가져오기
		일반적으로 per_cpu_ptr을 수행하여 workqueue_struct 구조체 cpu_pwqs(percpu)에 접근하여 cpu 주소를 읽어온다
		`__per_cpu_offset[n]`배열에 접근하여 offset을 알아내 주소를 연산
		last_pool = get_work_pool(work); 에서
			work_struct를 통해 get_work_pool을 불러와 worker_pool 구조체의 주소를 last_pool에 저장
			이 때 자료구조 흐름은 [[워크큐 자료구조 overall]] 에 명시됨
		워크를 이미 워커풀에서 실행한 적 있는지 검사
		워크를 이미 처리한 풀워크가 있으면 이를 통해 재사용
		find_worker_executing_work()를 통해 워크를 실행한 워커 스레드 구조체의 주소를 worker에 저장
		worker->current_pwq->wq에 저장된 워크큐 주소와 지금 큐잉하는 워크큐 주소가 동일하면 pool_workqueue 주소를 가져옴
		> 워크를 여러개의 워커 스레드에서 처리하지 못하게 제약을 부여
	3단계 : ftrace log 출력
		큐잉 출력
		if 조건문으로 실행중인 워커의 개수를 점검합니다
			풀워크큐에서 실행중인 워커의개수가 255를 넘었는지 여부에 따라 동작
				넘었을 경우 delayed_works라는 연결리스트에 등록후 처리
		active_work 출력
		실행중인 워커 수 증가
	4단계 : 워커풀에 워크의 연결 리스트를 등록하고 워커 스레드 깨우기
		pwq에는 pool_workqueue 구조체 주소를 담고 있고, work는 큐잉될 워크, worklist는 연결리스트
		insert_work() 를 호출하여 &pwq->pool->worklist 연결리스트에 work의 entry 등록
		insert_work내부에서 wake_up_worker()를 호출하여 워커 스레드를 깨운다

`__queue_work()`내부 함수
---
get_work_pool() 는 work_struct구조체의 data를 통해 워커 풀 주소를 반환하는 함수

구현부
%%kernel/workqueue.c%%
```
static struct worker_pool *get_work_pool(struct work_struct *work)
{
	unsigned long data = atomic_long_read(&work->data);
	int pool_id;

	assert_rcu_or_pool_mutex();

	if (data & WORK_STRUCT_PWQ)
		return ((struct pool_workqueue *)
			(data & WORK_STRUCT_WQ_DATA_MASK))->pool;

	pool_id = data >> WORK_OFFQ_POOL_SHIFT;
	if (pool_id == WORK_OFFQ_POOL_NONE)
		return NULL;

	return idr_find(&worker_pool_idr, pool_id);
}
```
data를 저장하고 WORK_STRUCT_PWQ(== 4)와 AND연산 결과가 true이면
	data와 0xFFFF_FFE0를 and연산한 결과를 pool_workqueue구조체로 캐스팅하여 pool_workqueue 구조체의 필드인 pool을 반환
data를 WORK_OFFQ_POOL_SHIFT(== 5) 만큼 시프트한 결과를 저장하여 pool_id로 워커풀의 주소를 읽어서 반환, worker_pool_idr에 각 노드별 워커풀이 등록되어있다.


insert_work()함수는 워커풀의 worklist에 work_struct의 entry를 저장, 스레드를 깨움

구현부
%%kernel/workqueue.c%%
```
static void insert_work(struct pool_workqueue *pwq, struct work_struct *work,
			struct list_head *head, unsigned int extra_flags)
{
	struct worker_pool *pool = pwq->pool;

	/* we own @work, set data and link */
	set_work_pwq(work, pwq, extra_flags);
	list_add_tail(&work->entry, head);
	get_pwq(pwq);

	smp_mb();

	if (__need_more_worker(pool))
		wake_up_worker(pool);
}
```
연결리스트에 entry를 연결하고, wake_up_worker를 호출하여 스레드를 깨움

wake_up_worker()구현부
%%kernel/workqueue.c%%
```
static void wake_up_worker(struct worker_pool *pool)
{
	struct worker *worker = first_idle_worker(pool);

	if (likely(worker))
		wake_up_process(worker->task);
}
```
워커풀에 등록된 idle 워커를 가져오고, 해당 워커를 깨운다(task_struct)


find_worker_executing_work() 함수는 기존에 실행된 워커의 주소를 반환하는 기능을 수행
%%kernel/workqueue.c%%
```
static struct worker *find_worker_executing_work(struct worker_pool *pool,
						 struct work_struct *work)
{
	struct worker *worker;

	hash_for_each_possible(pool->busy_hash, worker, hentry,
			       (unsigned long)work)
		if (worker->current_work == work &&
		    worker->current_func == work->func)
			return worker;

	return NULL;
}
```
pool의 busy_hash에는 실행중인 워커가 등록되어있다(배열)
이를 통해 워크와 워크핸들러를 비교하여 같다면 worker를 반환
	이미 워크를 실행한 워커가 있다면 해당 워커를 반환하는 동작

워크의 실행
---
워커가 깨어난 이후 워커 스레드 핸들러 함수 worker_thread() 함수가 실행
	내부에서 process_one_work()를 호출하여 워크핸들러를 호출한다

worker_thread() 함수
%%kernel/workqueue.c%%
```
static int worker_thread(void *__worker)
{
	struct worker *worker = __worker;
	struct worker_pool *pool = worker->pool;
...
do {
		struct work_struct *work =
			list_first_entry(&pool->worklist,
					 struct work_struct, entry);

		pool->watchdog_ts = jiffies;

		if (likely(!(*work_data_bits(work) & WORK_STRUCT_LINKED))) {
			/* optimization path, not strictly necessary */
			process_one_work(worker, work);
			if (unlikely(!list_empty(&worker->scheduled)))
				process_scheduled_works(worker);
		} else {
			move_linked_works(work, &worker->scheduled, NULL);
			process_scheduled_works(worker);
		}
	} while (keep_working(pool));
```
매개변수를 통해 pool 주소를 가져온다
keep_working 함수가 true를 반환하면 계속 실행
	이미 큐잉된 워크가 있으면 true를 반환하는 함수
워커풀의 연결리스트에서 워크 자료구조인 work_struct 주소를 읽어온다
현재 시각 정보(jiffies)를 워커풀의 watchgod_ts에 저장, 가장 최근 실행 시간 저장

work_struct의 data가 WORK_STRUCT_LINKED가 아닌지 점검, 일반적으로는 아님 > 이후 실행
process_one_work함수를 호출

%%kernel/workqueue.c%%
```
static bool keep_working(struct worker_pool *pool)
{
	return !list_empty(&pool->worklist) &&
		atomic_read(&pool->nr_running) <= 1;
}
```
연결리스트가 비어있는지 list_empty로 체크하고 비어있지 않으면 true를 반환


process_one_work() 함수
1단계 : 전처리
	워크의 entry를 워커풀 워크 리스트에서 해제
	워크의 실행상태 변환
2단계 : 워크 핸들러 실행
	ftrace의 workqueue_execute_start와 workqueue_excute_end
	워크 핸들러 호출

%%kernel/workqueue.c%%
```
static void process_one_work(struct worker *worker, struct work_struct *work)
__releases(&pool->lock)
__acquires(&pool->lock)
{
	struct pool_workqueue *pwq = get_work_pwq(work);
	struct worker_pool *pool = worker->pool;
	bool cpu_intensive = pwq->wq->flags & WQ_CPU_INTENSIVE;
	int work_color;
	struct worker *collision;
...
	collision = find_worker_executing_work(pool, work);
	if (unlikely(collision)) {
		move_linked_works(work, &collision->scheduled, NULL);
		return;
	}

	/* claim and dequeue */
	debug_work_deactivate(work);
	hash_add(pool->busy_hash, &worker->hentry, (unsigned long)work);
	worker->current_work = work;
	worker->current_func = work->func;
	worker->current_pwq = pwq;
	work_color = get_work_color(work);
...
	list_del_init(&work->entry);
...
	if (need_more_worker(pool))
		wake_up_worker(pool);
...
	set_work_pool_and_clear_pending(work, pool->id);
...
	trace_workqueue_execute_start(work);
	worker->current_func(work);
	trace_workqueue_execute_end(work);
...	
	if (unlikely(in_atomic() || lockdep_depth(current) > 0)) {
		pr_err("BUG: workqueue leaked lock or atomic: %s/0x%08x/%d\n"
		       "     last function: %pf\n",
		       current->comm, preempt_count(), task_pid_nr(current),
		       worker->current_func);
		debug_show_held_locks(current);
		dump_stack();
	}
...
```
collision = find_worker_executing_work(pool, work);
	워크가 다른 워커에서 현재 실행중인지 점검
	worker_pool의 busy_hash를 순회한다
조건 만족시 &collision->scheduled에 워크를 이동시키고 리턴

이후 hash_add를 통해 busy_hash에 &worker->hentry를 등록

워커의 각 필드에 work, work_func, pwq(pool_work_queue)를 등록

get_work_color를 통해 work_struct->data 중 color를 읽어온다

list_del_init을 통해 &work->entry를 wok_list라는 연결리스트에서 링크 해제

워커가 더 필요할 경우 워커스레드의 추가적인 깨우기

set_work_pool_and_clear_pending(work, pool->id);
	work의 data에 워커풀의 아이디를 설정하고 pending 비트를 클리어

worker->current_func를 통해 워크 핸들러 함수를 호출

현재 작업이 원자적 처리나 LOCL값이 0보다 크면 경고 출력