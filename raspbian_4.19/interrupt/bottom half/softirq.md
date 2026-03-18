발생 빈도가 높거나 빠르게 처리해야할 때 사용
	핸들러 호출 직후 실행
	동적으로 실행하도록 제공되는 [[tasklet]] 인터페이스가 존재

인터럽트가 아닐 때도 softirq 서비스 요청 가능능

네트워크 패킷처리나 고속 그래픽 처리, 스토리지(UFS) 드라이버를 구현
드라이버에서 요청한 동적커널 타이머들은 타이머 인터럽트 이후 softirq로 실행된다

softirq 서비스 요청을 받으면 이를 처리하는 과정

%%kernel/softirq.c%%
```
const char * const softirq_to_name[NR_SOFTIRQS] = {
	"HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "IRQ_POLL",
	"TASKLET", "SCHED", "HRTIMER", "RCU"
};
```
%%include/linux/interrupt.h%%
```
enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,
	BLOCK_SOFTIRQ,
	IRQ_POLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,
	HRTIMER_SOFTIRQ, /* Unused, but kept as tools rely on the
			    numbering. Sigh! */
	RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

	NR_SOFTIRQS
};
```

HI_SOFTIRQ
	높은 우선순위, TASKLET_HI로 적용
TIMER_SOFTIRQ
	동적 타이머
NET_TX_SOFTIRQ
	네트워크 패킷 송신용
NET_RX_SOFTIRQ
	네트워크 패킷 수신용
BLOCK_SOFTIRQ
	블록 디바이스에서 사용
IRQ_POLL_SOFTIRQ
	IRQ_POLL  연관 동작
TASKLET_SOFTIRQ
	일반 테스크릿용
SCHED_SOFTIRQ
	스케줄러에서 주로 사용
HRTIMER_SOFTIRQ
	하위호환성을 위한 용도
RCU_SOFTIRQ
	RCU 처리용
NR_SOFTIRQS


위의 서비스들은 부팅시 open_softirq() 함수를 통해 등록된다

요청시 raise_softirq() 함수를 호출
do_softirq() 함수에서 서비스 실행
	soft irq 서비스 핸들러를 호출하여 실행한다

실행흐름 [[interrupt 시 컨텍스트 흐름]]
1. 인터럽트 핸들러에서 soft irq 서비스를 요청
2. 인터럽트서비스 루틴 종료 시 irq_exit() 함수에서 softirq 유무 점검
3. 서비스 요청 확인 시 `__do_softirq()` 호출
4. 종료 시 다시 softirq 서비스 유무 다시 검토
	1. softirq 처리시간이 2ms 이상 or 10번 이상 수행했다면
			wakeup_softirq호출하여 ksoftirqd 프로세스를 깨움
			`__do_softirq 종료
5. ksoftirqd 프로세스가 마무리 못한 soft irq 서비스 실행

서비스 등록 예시
%%kernel/time/timer.c%%
```
void __init init_timers(void)
{
	init_timer_cpus();
	open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
}

```
TIMER_SOFTIRQ 서비스를 요청하면 run_timer_softirq() 함수를 호출해달라는 코드

open_softirq() 구현부
%%kernel/softirq.c%%
```
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```
정수 nr에 해당하는 softirq_vec 배열 원소의 action 필드에 action 할당(함수포인터)

softirq_vec는 10개의 함수 포인터로 구성된 배열로 `__do_softirq()`에서 참조

인터럽트 도중 raise_softirq() 를 호출
	`__raise_softirq_irqoff`
		or_softirq_pending
			`irq_stat[cpu].__sodtirq_pending`에 특정 비트 활성화(percpu타입변수)

서비스 요청
---
raise_softirq() 구현부
%%kernel/softirq.c%%
```
void raise_softirq(unsigned int nr)
{
	unsigned long flags;

	local_irq_save(flags);
	raise_softirq_irqoff(nr);
	local_irq_restore(flags);
}
```
인터럽트를 비활성화
softirq 서비스의 정수형 인덱스 nr을 인자로 raise_softirq_irqoff()호출

raise_softirq_irqoff()구현부
%%kernel/softirq.c%%
```
inline void raise_softirq_irqoff(unsigned int nr)
{
	__raise_softirq_irqoff(nr);

	if (!in_interrupt())
		wakeup_softirqd();
}
```
`__raise_softirq_irqoff()`호출
인터럽트 컨택스트가 아닐 시 ksoftirqd 스레드를 깨운다

`__raise_softirq_irqoff()`구현부
%%kernel/softirq.c%%
```
void __raise_softirq_irqoff(unsigned int nr)
{
	trace_softirq_raise(nr);
	or_softirq_pending(1UL << nr);
}
```
trace 로그 출력
nr을 비트연산한 결과값을 or_softirq_pending에 전달


or_softirq_pending() 구현부
%%include/linux/interrupt.h%%
```
#ifndef local_softirq_pending_ref
#define local_softirq_pending_ref irq_stat.__softirq_pending
#endif

#define local_softirq_pending()	(__this_cpu_read(local_softirq_pending_ref))
#define set_softirq_pending(x)	(__this_cpu_write(local_softirq_pending_ref, (x)))
#define or_softirq_pending(x)	(__this_cpu_or(local_softirq_pending_ref, (x)))
```
결과적으로 or_softirq_pending은
`irq_stat.__softirq_pending |= x;` 으로 치환

즉 irq_stat변수는 cpu별로 지정한 `__softirq_pending`변수에 (1<<softirq서비스 아이디)를 수행한 결과를 저장

local_softirq_pending()을 통해 한 비트라도 설정되어 있으면 true 반환하여 softirq 요청 여부 점검
	인터럽트 핸들러 이후 irq_exit
	ksoftirqd 스레드 핸들러 함수인 run_ksoftirqd()


점검부 코드
%%kernel/softirq.c%%
```
void irq_exit(void)
{
...
	if (!in_interrupt() && local_softirq_pending())
		invoke_softirq();
```
인터럽트 컨택스트가 아니고, softirq 요청이 있다면 유발
```
static void run_ksoftirqd(unsigned int cpu)
{
	local_irq_disable();
	if (local_softirq_pending()) {
		__do_softirq();
```
요청이 있다면 실행

실행
---
[[interrupt 시 컨텍스트 흐름]]
`__handle_domain_irq()`에서 irq_exit() 호출

irq_exit()구현부
%%kernel/softirq.c%%
```
void irq_exit(void)
{
...
	account_irq_exit_time(current);
	preempt_count_sub(HARDIRQ_OFFSET);
	if (!in_interrupt() && local_softirq_pending())
		invoke_softirq();
```
preempt_count_sub(HARDIRQ_OFFSET);를 통해 인터럽트 컨택스트 해제를 명시
softirq 서비스 요청을 점검

invoke_softirq() 구현부
%%kernel/softirq.c%%
```
static inline void invoke_softirq(void)
{
	if (ksoftirqd_running(local_softirq_pending()))
		return;

	if (!force_irqthreads) {
#ifdef CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK
		
		__do_softirq();
#else
		
		do_softirq_own_stack();
#endif
	} else {
		wakeup_softirqd();
	}
}
```
ksoftirqd가 진행중인데 요청을 했다면 return
	ksoftirqd에서 softirq를 중복으로 처리 가능
force_irqthreads가 설정되어있으면 ksoftirqd를 깨운다
	이 변수는 부팅 전 부트로터에서 threadirqs를 커맨드라인으로 전달하면 설정
				???????????????
CONFIG_HAVE_IRQ_EXIT_ON_IRQ_STACK가 라즈비안에서는 비활성화됨

do_softirq_own_stack()는 `__do_softirq()`를 바로 호출


`__do_softirq()`
---
%%kernel/softirq.c%%
```
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
	unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
	unsigned long old_flags = current->flags;
	int max_restart = MAX_SOFTIRQ_RESTART;
	struct softirq_action *h;
	bool in_hardirq;
	__u32 pending;
	int softirq_bit;
...
	pending = local_softirq_pending();
	account_irq_enter_time(current);

	__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
	in_hardirq = lockdep_softirq_start();

restart:
	/* Reset the pending bitmask before enabling irqs */
	set_softirq_pending(0);

	local_irq_enable();

	h = softirq_vec;

	while ((softirq_bit = ffs(pending))) {
		unsigned int vec_nr;
		int prev_count;

		h += softirq_bit - 1;

		vec_nr = h - softirq_vec;
		prev_count = preempt_count();

		kstat_incr_softirqs_this_cpu(vec_nr);

		trace_softirq_entry(vec_nr);
		h->action(h);
		trace_softirq_exit(vec_nr);
...
		h++;
		pending >>= softirq_bit;
	}

	rcu_bh_qs();
	local_irq_disable();

	pending = local_softirq_pending();
	if (pending) {
		if (time_before(jiffies, end) && !need_resched() &&
		    --max_restart)
			goto restart;

		wakeup_softirqd();
	}
...
```

unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
	시간정보를 담은 jiffies에 msecs_to_jiffies(2) 매크로를 통해 2ms를 1/HZ단위 상수로 치환하 종료시간을 지정 > `__do_softirq()`의 제한시간
int max_restart = MAX_SOFTIRQ_RESTART; == 10
	softirq의 연속 수행시 restart 마다 max_restart 감소, 0이 되면 종료
	전역변수는 지역변수에 저장후 연산해야 동기화 문제를 피하기 용이하다

set_softirq_pending(0);
	`__softirq_pending`필드를 0으로 초기화
h = softirq_vec;
while ((softirq_bit = ffs(pending))) { ... h += softirq_bit - 1;
	ffs를 통해 계산된 비트를 h = softirq_vec에 더하여 
	softirq 서비스 핸들러함수의 주소를 연산

trace_softirq_entry(vec_nr);
h->action(h);
trace_softirq_exit(vec_nr);
	연산된 주소를 바탕으로 softirq 서비스 핸들러 호출, ftrace log

pending = local_softirq_pending();
	softirq 요청 여부 다시 확인
		요청 존재시 시간, 횟수를 연산하여 restart로 다시 갈지 결정
		시간, 횟수 초과시 ksoftirqd 깨우기   wakeup_softirqd()

wakeup_softirqd() 구현부
%%kernel/softirq.c%%
```
static void wakeup_softirqd(void)
{
	/* Interrupts are disabled: no need to stop preemption */
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

	if (tsk && tsk->state != TASK_RUNNING)
		wake_up_process(tsk);
}
```

`__this_cpu_read(ksoftirqd);
	ksoftirqd의 태스크 디스크립터를 가져옴
	조건 만족시 깨운다

깨어난 경우 ksoftirqd 핸들러 함수인 run_ksoftirqd() 호출

 run_ksoftirqd() 구현부
 %%kernel/softirq.c%%
 ```
 static void run_ksoftirqd(unsigned int cpu)
{
	local_irq_disable();
	if (local_softirq_pending()) {
		
		__do_softirq();
		local_irq_enable();
		cond_resched();
		return;
	}
	local_irq_enable();
}
 ```
요청 점검후 `__do_softirq();`


ksoftirqd thread
---
percpu 타입 프로세스
softirq 서비스를 프로세스 레벨에서 처리할 수 있도록 지원

%%kernel/softirq.c%%
```
static __init int spawn_ksoftirqd(void)
{
	cpuhp_setup_state_nocalls(CPUHP_SOFTIRQ_DEAD, "softirq:dead", NULL,
				  takeover_tasklets);
	BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

	return 0;
}
```
부팅 시 smpboot_register_percpu_thread() 실행 시 생성된다

ksoftirqd 스레드의 선언부
%%kernel/softirq.c%%
```
static struct smp_hotplug_thread softirq_threads = {
	.store			= &ksoftirqd,
	.thread_should_run	= ksoftirqd_should_run,
	.thread_fn		= run_ksoftirqd,
	.thread_comm		= "ksoftirqd/%u",
};
```
프로세스가 실행되면 run_ksoftirqd 함수가 실행되는 것 확인 가능
	percpu타입 스레드는 smp 핫플러그 스레드로 등록되어 실행된다 [[others]]

`__do_softirq()`에서 서비스 실행한 후
	`__do_softirq()`의 실행시간이 MAX_SOFTIRQ_TIME을 넘은 경우
인터럽트가 아닌 상황에서 softirq서비스를 요청한 경우
	rraise_softirq_irqoff() 에서 인터럽트 컨택스트 여부를 점검하고 걸러낸다

ksoftirqd_should_run() 에서 실행여부를 점검한다
%%kernel/softirq.c%%
```
static int ksoftirqd_should_run(unsigned int cpu)
{
	return local_softirq_pending();
}
```
요청있을 시 true 반환


softirq 컨택스트 활성화, 종료 시점
---

`__do_softirq() 에서 실행흐름이 제어된다`
```
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
...
	__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
	in_hardirq = lockdep_softirq_start();

restart:
...
	while ((softirq_bit = ffs(pending))) {
...
		trace_softirq_entry(vec_nr);
		h->action(h);
		trace_softirq_exit(vec_nr);
...
	lockdep_softirq_end(in_hardirq);
	account_irq_exit_time(current);
	__local_bh_enable(SOFTIRQ_OFFSET);
	WARN_ON_ONCE(in_interrupt());
	current_restore_flags(old_flags, PF_MEMALLOC);
}
...
```

`__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET);
	softirq 컨택스트 활성화
`	__local_bh_enable(SOFTIRQ_OFFSET);`
	softirq 컨택스트 비활성화

thread_info 구조체의 preempt_count 필드를 제어
	0x100을 더하고 뺀다
		`#define SOFTIRQ_OFFSET  (1UL << SOFTIRQ_SHIFT)`


`__local_bh_disable_ip()` 구현부
%%kernel/softirq.c%%
```
void __local_bh_disable_ip(unsigned long ip, unsigned int cnt)
{
	unsigned long flags;

	WARN_ON_ONCE(in_irq());

	raw_local_irq_save(flags);
	__preempt_count_add(cnt);
```
cnt를 인자로 `__preempt_count_add(cnt);`에 전달

 `__preempt_count_add()`구현부
%%include/asm-generic/preempt.h%%
```
static __always_inline void __preempt_count_add(int val)
{
	*preempt_count_ptr() += val;
}
```

`*preempt_count_ptr() 구현부`
%%include/asm-generic/preempt.h%%
```
static __always_inline volatile int *preempt_count_ptr(void)
{
	return &current_thread_info()->preempt_count;
}
```
current_thread_info()를 통해 스택 최상단 주소 thread_info 구조체의preempt_count 구조체에 접근한다

current_thread_info() 구현부
%%arch/arm/include/asm/thread_info.h%%
```
static inline struct thread_info *current_thread_info(void)
{
	return (struct thread_info *)
		(current_stack_pointer & ~(THREAD_SIZE - 1));
}
```

더 자세한 사항 >> [[thread_info]]


`	__local_bh_enable(SOFTIRQ_OFFSET);` 구현부
%%kernel/softirq.c%%
```
static void __local_bh_enable(unsigned int cnt)
{
	lockdep_assert_irqs_disabled();

	if (preempt_count() == cnt)
		trace_preempt_on(CALLER_ADDR0, get_lock_parent_ip());

	if (softirq_count() == (cnt & SOFTIRQ_MASK))
		trace_softirqs_on(_RET_IP_);

	__preempt_count_sub(cnt);
}
```

`__preempt_count_sub(cnt);` 구현부
%%%%
```
static __always_inline void __preempt_count_sub(int val)
{
	*preempt_count_ptr() -= val;
}
```
