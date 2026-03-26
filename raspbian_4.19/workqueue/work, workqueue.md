워크큐는 기본적으로 7개지원, 부팅 과정에서 생성
디바이스 드라이버에서 새 워크큐를 생성하여 사용할 수 있다

워크큐의 생성
---

%%include/linux/workqueue.h%%
```
#define alloc_workqueue(fmt, flags, max_active, args...)		\
({									\
	static struct lock_class_key __key;				\
	const char *__lock_name;					\
									\
	__lock_name = "(wq_completion)"#fmt#args;			\
									\
	__alloc_workqueue_key((fmt), (flags), (max_active),		\
			      &__key, __lock_name, ##args);		\
})
```

fmt > 워크큐의 이름을 지정하여 workqueue_struct 의 name에 저장한다
flags > 워크큐의 속성정보를 저장
	%%include/linux/workqueue.h%%
max_active > workqueue_struct의 saved_max_active에 저장

`__alloc_workqueue_key()`를 호출하여 워크큐의 생성을 이어감
	`__alloc_workqueue_key()`에서 alloc_and_link_pwqs() 를 수행
%%kernel/workqueue.c%%
```
static int alloc_and_link_pwqs(struct workqueue_struct *wq)
{
	bool highpri = wq->flags & WQ_HIGHPRI;
	int cpu, ret;
...
		for_each_possible_cpu(cpu) {
			struct pool_workqueue *pwq =
				per_cpu_ptr(wq->cpu_pwqs, cpu);
			struct worker_pool *cpu_pools =
				per_cpu(cpu_worker_pools, cpu);

			init_pwq(pwq, wq, &cpu_pools[highpri]);

```

 alloc_and_link_pwqs()에서 flags가 WQ_HIGHPRI 이면 `cpu_worker_pool[1]`에 위치한 워크풀을 워크큐에 할당한다 >> 해당 cpu에서 우선순위가 높다는 말과 동일

생성부
workqueue_init_early()
%%kernel/workqueue.c%%
```
int __init workqueue_init_early(void)
{
	int std_nice[NR_STD_WORKER_POOLS] = { 0, HIGHPRI_NICE_LEVEL };
	int hk_flags = HK_FLAG_DOMAIN | HK_FLAG_WQ;
	int i, cpu;
...
	system_wq = alloc_workqueue("events", 0, 0);
	system_highpri_wq = alloc_workqueue("events_highpri", WQ_HIGHPRI, 0);
	system_long_wq = alloc_workqueue("events_long", 0, 0);
	system_unbound_wq = alloc_workqueue("events_unbound", WQ_UNBOUND,
					    WQ_UNBOUND_MAX_ACTIVE);
	system_freezable_wq = alloc_workqueue("events_freezable",
					      WQ_FREEZABLE, 0);
	system_power_efficient_wq = alloc_workqueue("events_power_efficient",
					      WQ_POWER_EFFICIENT, 0);
	system_freezable_power_efficient_wq = alloc_workqueue("events_freezable_power_efficient",
					      WQ_FREEZABLE | WQ_POWER_EFFICIENT,
					      0);
	BUG_ON(!system_wq || !system_highpri_wq || !system_long_wq ||
	       !system_unbound_wq || !system_freezable_wq ||
	       !system_power_efficient_wq ||
	       !system_freezable_power_efficient_wq);

	return 0;
}
```
7가지 워크큐를 생성, 점검
	system_wq 는 시스템 워크큐라고도 불림
	system_highpri_wq는 시스템 워크큐보다 우선순위가 높다
		workqueue_struct 의 flags 는 WQ_HIGHPRI 로 설정
	system_long_wq
		좀 더 오래걸리는 작업
	system_unbound_wq
		percpu 워커가 아닌 `wq->numa_pwq_tbl[node]` 에 지정된 워커풀 사용
		workqueue_struct의 flags와 and연산하여 events_unbound 여부 판단
	system_freezable_wq
		얼릴 수 있는 프로세스를 처리할 때 freeze_workqueues_begin()에서 수행
			자세한 내용은 책을 넘어섬, 책 1권 p554
	나머지 2개는 절전목적으로 사용하는 워크큐


워크
---

워크큐의 실행단위로 각 워크는 부팅단계에서 초기화된다
워크의 실행을 원할 시 워크큐에 큐잉하고, 워커스레드를 깨우면 실행

워크 자료구조
%%include/linux/workqueue.h%%
```
struct work_struct {
	atomic_long_t data;
	struct list_head entry;
	work_func_t func;
#ifdef CONFIG_LOCKDEP
	struct lockdep_map lockdep_map;
#endif
};
```
rpi에서 CONFIG_LOCKDEP은 비활성화
atomic_long_t data
	워크의 실행상태를 표현 > 제어시 활용
		워크 초기화시 : 0xFFFFFFFE0
		워크큐에 큐잉시 : WORK_STRUCT_PENDING_BIT(0x1)
		워크를 실행한 적 있다면 pool_workqueue 구조체의 주소가 저장됨
struct list_head entry
	연결리스트로 큐잉하면 worker_pool 구조체 중 연결리스트인 worklist에 등록된다
	즉 worker_pool 구조체의 worklist는 워크의 entry 주소를 저장
work_func_t func
	워크 핸들러 함수의 주소 > 워커스레드가 실행되면서 워크 핸들러함수 호출

워크의 초기화
---
부팅시 수행
	INIT_WORK()     OR      DECLARE_WORK()

실행 예시
%%drivers/tty/tty_buffer.c%%
```
void tty_buffer_init(struct tty_port *port)
{
	struct tty_bufhead *buf = &port->buf;

	mutex_init(&buf->lock);
	tty_buffer_reset(&buf->sentinel, 0);
	buf->head = &buf->sentinel;
	buf->tail = &buf->sentinel;
	init_llist_head(&buf->free);
	atomic_set(&buf->mem_used, 0);
	atomic_set(&buf->priority, 0);
	INIT_WORK(&buf->work, flush_to_ldisc);
	buf->mem_limit = TTYB_DEFAULT_MEM_LIMIT;
}
```
&buf->work는 work_struct이고, flush_to_ldisc는 워크실행시 핸들러

INIT_WORK()구현부
%%include/linux/workqueue.h%%
```
#define __INIT_WORK(_work, _func, _onstack)				\
	do {								\
		__init_work((_work), _onstack);				\
		(_work)->data = (atomic_long_t) WORK_DATA_INIT();	\
		INIT_LIST_HEAD(&(_work)->entry);			\
		(_work)->func = (_func);				\
	} while (0)
#endif

#define INIT_WORK(_work, _func)						\
	__INIT_WORK((_work), (_func), 0)
...
#define WORK_DATA_INIT()	ATOMIC_LONG_INIT((unsigned long)WORK_STRUCT_NO_POOL)
```

`__init_work()`함수는 CONFIG_DEBUG_OBJECTS가 활성화 되어야 수행
	RPI에서는 비활성화
WORK_STRUCT_NO_POOL == 0xFFFF_FFE0
	자세한 계산과정은 책 1권 561p
work_struct의 entry list의 초기화
워크 핸들러 함수의 주소를 지정



DECLARE_WORK() 구현부
%%include/linux/workqueue.h%%
```
#define DECLARE_WORK(n, f)						\
	struct work_struct n = __WORK_INITIALIZER(n, f)
...
#define __WORK_INITIALIZER(n, f) {					\
	.data = WORK_DATA_STATIC_INIT(),				\
	.entry	= { &(n).entry, &(n).entry },				\
	.func = (f),							\
	__WORK_INIT_LOCKDEP_MAP(#n, &(n))				\
	}
...
#define WORK_DATA_STATIC_INIT()	\
	ATOMIC_LONG_INIT((unsigned long)(WORK_STRUCT_NO_POOL | WORK_STRUCT_STATIC))
```
WORK_DATA_STATIC_INIT() 은 두 플래그를 OR연산 수행
	WORK_STRUCT_NO_POOL == 0xFFFF_FFE0
	WORK_STRUCT_STATIC == 0

즉 어느 함수를 쓰더라도 data에 0xFFFF_FFE0, func에 워크 핸들러 저장을 수행한다
0xFFFF_FFE0는 초기화 후 아직 실행하지 않았다는 의미이다


큐잉과정 [[queuing, acting]]