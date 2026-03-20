동적으로 softirq 서비스를 사용하는 인터페이스
태스크릿은 인터럽트 발생빈도가 높거나 인터럽트 후반부의 빠른 처리가 요구될 때 사용
디바이스 드라이버에서 softirq를 활용하여 인터럽트를 처리하고 싶을 때 사용

softirq 서비스 별 핸들러 함수들의 배열softirq_vec 
	tasklet 서비스 헨들러 함수 == tasklet_action()

tasklet_action함수는 각 테스크릿에 등록된 tasklet 핸들러 함수를 호출해주는 인터페이스기능

절차
	인터럽트 핸들러에서 tasklet_schedule()호출
	tasklet_schedule() 내붓에서 raise_softirq_irqoff() 호출하여 서비스 요청
	softirq 실행시 서비스 핸들러인 tasklet_action() 실행
	`__do_softirq()`에서 처리 못하면 ksoftirqd 스레드로 해결


tasklet_struct 선언부
%%include/linux/interrupt.h%%
```
struct tasklet_struct
{
	struct tasklet_struct *next;
	unsigned long state;
	atomic_t count;
	void (*func)(unsigned long);
	unsigned long data;
};
```

필드
	`struct tasklet_struct *next`
		다음 테스크릿의 포인터
	unsigned long state
		태스크릿의 세부 상태 정보 저장 필드
		`TASKLET_STATE_SCHED,	/* Tasklet is scheduled for execution */
			요청 후 실행하지 않은 상태
		`TASKLET_STATE_RUN	/* Tasklet is running (SMP only) */
			태스크릿 핸들러 실행중
	atomic_t count
		실행여부를 판단하는 필드, 0이어야만 실행
	`void (*func)(unsigned long)`
		태스크릿 핸들러 함수 주소
	unsigned long data
		태스크릿 핸들러 함수에 전달될 매개변수

초기화
---
DECLARE_TASKLET  or  DECLARE_TASKLET_DISABLED  로 초기화
	컴파일 타임에 등록된다
%%include/linux/interrupt.h%%
```
#define DECLARE_TASKLET(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(0), func, data }

#define DECLARE_TASKLET_DISABLED(name, func, data) \
struct tasklet_struct name = { NULL, 0, ATOMIC_INIT(1), func, data }
```
name >> tasklet_struct 타입
func >> 태스크릿 핸들러 함수
data >> 태스크릿 핸들러 함수에 전달되는 매개변수

atomic_init(0) == 0 을 count에 넣는다
DECLARE_TASKLET_DISABLED 으로 초기화시 기본 설정 비활성화
tasklet_enable로 활성화 할 수 있다
%%include/linux/interrupt.h%%
```
static inline void tasklet_enable(struct tasklet_struct *t)
{
	smp_mb__before_atomic();
	atomic_dec(&t->count);
}
```

tasklet_init()함수로 초기화
%%kernel/softirq.c%%
```
void tasklet_init(struct tasklet_struct *t,
		  void (*func)(unsigned long), unsigned long data)
{
	t->next = NULL;
	t->state = 0;
	atomic_set(&t->count, 0);
	t->func = func;
	t->data = data;
}
```
tasklet_struct에 각 필드 대입


실행흐름
---
1. 인터럽트 핸들러에서 tasklet_schedule() 호출   {태스크릿 스케줄링}
	raise_softirq_irqoff() 호출
2. `__do_softirq()`에서 tasklet_action 호출
	태스크릿 핸들러 함수 호출
3. 실행 중 실행시간을 체크하고 초과시 ksoftirqd 깨우기


tasklet_schedule()은 tasklet_struct의 state를 TASKLET_STATE_SCHED로 바꾸고
`__tasklet_schedule() 호출

`__tasklet_schedule() 구현부
%%kernel/softirq.c%%
```
void __tasklet_schedule(struct tasklet_struct *t)
{
	__tasklet_schedule_common(t, &tasklet_vec,
				  TASKLET_SOFTIRQ);
}
```
tasklet_vec(연결리스트)과 TASKLET_SOFTIRQ 을 `__tasklet_schedule_common`으로 전달

`__tasklet_schedule_common()` 구현부
%%kernel/softirq.c%%
```
static void __tasklet_schedule_common(struct tasklet_struct *t,
				      struct tasklet_head __percpu *headp,
				      unsigned int softirq_nr)
{
	struct tasklet_head *head;
	unsigned long flags;

	local_irq_save(flags);
	head = this_cpu_ptr(headp);
	t->next = NULL;
	*head->tail = t;
	head->tail = &(t->next);
	raise_softirq_irqoff(softirq_nr);
	local_irq_restore(flags);
}
```

인터럽트 비활성화
실행요청할 태스크릿을 tasklet_vec 연결리스트의 tail에 추가
	tasklet_vec은 percpu 타입이다
raise_softirq_irqoff(softirq_nr) 를 호출하여 `__do_softirq()` 수행
	[[softirq]]
인터럽트 활성화

`__do_softirq()` 에서 tasklet_action 함수 호출

tasklet_action() 구현부
%%kernel/softirq.c%%
```
static __latent_entropy void tasklet_action(struct softirq_action *a)
{
	tasklet_action_common(a, this_cpu_ptr(&tasklet_vec), TASKLET_SOFTIRQ);
}
```
a >> struct softirq_action 타입
this_cpu_ptr(&tasklet_vec) >> tasklet_vec전역변수에서 percpu 오프셋 적용한 주소
TASKLET_SOFTIRQ >> 태스크릿 플래그

tasklet_action_common() 구현부
%%kernel/softirq.c%%
```
static void tasklet_action_common(struct softirq_action *a,
				  struct tasklet_head *tl_head,
				  unsigned int softirq_nr)
{
	struct tasklet_struct *list;

	local_irq_disable();
	list = tl_head->head;
	tl_head->head = NULL;
	tl_head->tail = &tl_head->head;
	local_irq_enable();

	while (list) {
		struct tasklet_struct *t = list;

		list = list->next;

		if (tasklet_trylock(t)) {
			if (!atomic_read(&t->count)) {
				if (!test_and_clear_bit(TASKLET_STATE_SCHED,
							&t->state))
					BUG();
				t->func(t->data);
				tasklet_unlock(t);
				continue;
			}
			tasklet_unlock(t);
		}

		local_irq_disable();
		t->next = NULL;
		*tl_head->tail = t;
		tl_head->tail = &t->next;
		__raise_softirq_irqoff(softirq_nr);
		local_irq_enable();
	}
}
```

FIFO 와 유사하게 접근한다

tasklet_struct 체크
if (!atomic_read(&t->count)) 에서 count가 0인지 체크
	state가 TASKLET_STATE_SCHED가 아니면 커널 패닉

t->func(t->data) >> 태스크릿 핸들러 함수를 호출하는 코드