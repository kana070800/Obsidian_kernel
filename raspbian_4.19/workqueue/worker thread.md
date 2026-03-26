kworker/* 형식의 이름을 가진 프로세스
미리 생성해 놓고 워크 실행 요청시 처리

1. 워커 스레드를 생성
	create_worker()를 통해 워커 스레드 생성
2. 휴면 상태
	휴면 상태에서 다른 드라이버가 자신을 깨워주길 기다림
3. 실행
	워크를 큐잉한 후 깨어나면, 스래드 핸들러인 worker_thread() 실행
4. 소멸

구조체 선언부
%%kernel/workqueue_internal.h%%
```
struct worker {
	/* on idle list while idle, on busy hash table while busy */
	union {
		struct list_head	entry;	/* L: while idle */
		struct hlist_node	hentry;	/* L: while busy */
	};

	struct work_struct	*current_work;	/* L: work being processed */
	work_func_t		current_func;	/* L: current_work's fn */
	struct pool_workqueue	*current_pwq; /* L: current_work's pwq */
	struct list_head	scheduled;	/* L: scheduled works */

	/* 64 bytes boundary on 64bit, 32 on 32bit */

	struct task_struct	*task;		/* I: worker task */
	struct worker_pool	*pool;		/* A: the associated pool */
						/* L: for rescuers */
	struct list_head	node;		/* A: anchored at pool->workers */
						/* A: runs through worker->node */

	unsigned long		last_active;	/* L: last active timestamp */
	unsigned int		flags;		/* X: flags */
	int			id;		/* I: worker id */

	char			desc[WORKER_DESC_LEN];

	/* used only by rescuers to point to the target workqueue */
	struct workqueue_struct	*rescue_wq;	/* I: the workqueue to rescue */
};
```
`struct work_struct *current_work
	work_struct 구조체로 현재 실행하려는 워크
work_func_t current_func
	실행하려는 워크 핸들러의 주소
`struct task_struct *task`
	워커 스레드의 태스크 디스크립터의 주소
struct list_head node
	워커 풀에 등록된 연결 리스트

부팅시 워커 스레드 생성 코드
%%kernel/workqueue.c%%
```
int __init workqueue_init(void)
{
	struct workqueue_struct *wq;
	struct worker_pool *pool;
	int cpu, bkt;
	
	wq_numa_init();
	
	mutex_lock(&wq_pool_mutex);
...
	mutex_unlock(&wq_pool_mutex);
	
	/* create the initial workers */
	for_each_online_cpu(cpu) {
		for_each_cpu_worker_pool(pool, cpu) {
			pool->flags &= ~POOL_DISASSOCIATED;
			BUG_ON(!create_worker(pool));
		}
	}
...
```
for_each_online_cpu에서 각 cpu의 워커풀에 접근하여 풀 노드 정보를 저장
	각 풀마다 create_worker를 호출하여 워커를 생성

create_worker()
	워커풀의 아이디 읽어오기
	워커 스레드의 이름을 지정하여 스레드 생성요청
	워커풀에 스레드 등록
	워커 정보를 갱신하고 생성된 워커 스레드 깨우기

구현부
%%kernel/workqueue.c%%
```
static struct worker *create_worker(struct worker_pool *pool)
{
	struct worker *worker = NULL;
	int id = -1;
	char id_buf[16];

	/* ID is needed to determine kthread name */
	id = ida_simple_get(&pool->worker_ida, 0, 0, GFP_KERNEL);
	if (id < 0)
		goto fail;

	worker = alloc_worker(pool->node);
	if (!worker)
		goto fail;

	worker->id = id;

	if (pool->cpu >= 0)
		snprintf(id_buf, sizeof(id_buf), "%d:%d%s", pool->cpu, id,
			 pool->attrs->nice < 0  ? "H" : "");
	else
		snprintf(id_buf, sizeof(id_buf), "u%d:%d", pool->id, id);

	worker->task = kthread_create_on_node(worker_thread, worker, pool->node,
					      "kworker/%s", id_buf);
	if (IS_ERR(worker->task))
		goto fail;

	set_user_nice(worker->task, pool->attrs->nice);
	kthread_bind_mask(worker->task, pool->attrs->cpumask);

	/* successful, attach the worker to the pool */
	worker_attach_to_pool(worker, pool);

	/* start the newly created worker */
	spin_lock_irq(&pool->lock);
	worker->pool->nr_workers++;
	worker_enter_idle(worker);
	wake_up_process(worker->task);
	spin_unlock_irq(&pool->lock);

	return worker;

fail:
	if (id >= 0)
		ida_simple_remove(&pool->worker_ida, id);
	kfree(worker);
	return NULL;
}
```
snprintf 함수에서 워커 스레드의 이름을 지정한 후 프로세스 생성 요청