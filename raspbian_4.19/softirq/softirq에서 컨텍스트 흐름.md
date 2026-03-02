
irq_exit() 함수가 softirq의 시작점
	irq_exit > do_softirq() 호출
	인터럽트 헨들링 직후 시작

do_softirq() 구현부
%%kernel/softirq.c%%
```
asmlinkage __visible void __softirq_entry __do_softirq(void)
{
...
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
	}
...
	lockdep_softirq_end(in_hardirq);
	account_irq_exit_time(current);
	__local_bh_enable(SOFTIRQ_OFFSET);
...
```
`__local_bh_disable_ip(_RET_IP_, SOFTIRQ_OFFSET)`
	thread_info 의 preempt_count 에 접근해 SOFTIRQ_OFFSET
	`__preempt_count_add(cnt)` 수행
softirq 수행
`__local_bh_enable(SOFTIRQ_OFFSET)`
	thread_info 의 preempt_count 에 접근해 SOFTIRQ_OFFSET
	`__preempt_count_sub(cnt)` 수행


preempt_count 필드가 0 이면 프로세스 선점될 수 있는 조건
선점 스케줄링 코드
%%linux/arch/arm/kernel/entry-armv.S%%
```
__irq_svc:
	svc_entry
	irq_handler

#ifdef CONFIG_PREEMPT
	ldr	r8, [tsk, #TI_PREEMPT]		@ get preempt count
	ldr	r0, [tsk, #TI_FLAGS]		@ get flags
	teq	r8, #0				@ if preempt count != 0
	movne	r0, #0				@ force flags to 0
	tst	r0, #_TIF_NEED_RESCHED
	blne	svc_preempt
#endif
```
ldr로 preempt 값을 가져오고 0이고, flags가 특정 비트 포함시 svc_preempt로 이동
	선점 스케쥴링 수행