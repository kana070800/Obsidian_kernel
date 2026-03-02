
인터럽트 컨텍스트인지 확인하는 방법
	in_interrupt() 제공

실행흐름
	interrupt 발생
	`__irq_svc` 레이블 실행
	인터럽트 제어함수 호출
		bcm_arm_irqchip_handle_irq()
		`__handle_domain_irq()`
	`__handle_domain_irq()` 에서 irq_enter()
		여기서 thread_info 의 preempt_count 필드 변경 > HARDRIQ_OFFSET 더하기
	인터럽트 헨들러 함수 호출
	`__handle_domain_irq()` 에서 irq_exit()
		여기서 thread_info 의 preempt_count 필드 변경 > HARDRIQ_OFFSET 빼기

`__irq_enter` 구현부
%%include/linux/hardirq.h%%
```
#define __irq_enter()					\
	do {						\
		account_irq_enter_time(current);	\
		preempt_count_add(HARDIRQ_OFFSET);	\
		trace_hardirq_enter();			\
	} while (0)
```
%%include/linux/preempt.h%%
```
#define preempt_count_add(val)	__preempt_count_add(val)
```
thread_info 의 preempt_count 필드 변경 > HARDRIQ_OFFSET 더하기

irq_exit() 구현부
%%kernel/softirq.c%%
```
void irq_exit(void)
{
#ifndef __ARCH_IRQ_EXIT_IRQS_DISABLED
	local_irq_disable();
#else
	lockdep_assert_irqs_disabled();
#endif
	account_irq_exit_time(current);
	preempt_count_sub(HARDIRQ_OFFSET);
...
```
thread_info 의 preempt_count 필드 변경 > HARDRIQ_OFFSET 더하기