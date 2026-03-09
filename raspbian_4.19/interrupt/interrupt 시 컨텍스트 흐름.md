
인터럽트 컨텍스트인지 확인하는 방법
	in_interrupt() 제공 [[raspbian_4.19/interrupt/information|information]]

실행흐름 [[thread_info]] 의 preempt_count 가 관여
	interrupt 발생
	vertor_irq 레이블 실행
	`__irq_svc or __irq_usr` 레이블 실행
	실행중인 프로세스의 레지스터 세트를 프로세스 스텍에 저장
	인터럽트 제어함수 호출
		bcm_arm_irqchip_handle_irq()
		`__handle_domain_irq()`
	`__handle_domain_irq()` 에서 irq_enter()
		여기서 thread_info 의 preempt_count 필드 변경 > HARDRIQ_OFFSET 더하기
	>>>>인터럽트 헨들러 함수 호출
	`__handle_domain_irq()` 에서 irq_exit()
		여기서 thread_info 의 preempt_count 필드 변경 > HARDRIQ_OFFSET 빼기


objdump를 통해 확인한 코드
%%arch/arm/kernel/entry-common.S%%
```
Disassembly of section .vectors:

ffff0000 <__init_begin+0x7f3f0000>:
ffff0000:	ea0003ff 	b	ffff1004 <vector_rst>
ffff0004:	ea000465 	b	ffff11a0 <vector_und>
ffff0008:	e59ffff0 	ldr	pc, [pc, #4080]	; ffff1000 <__crc_lock_two_nondirectories+0x378d7>
ffff000c:	ea000443 	b	ffff1120 <vector_pabt>
ffff0010:	ea000422 	b	ffff10a0 <vector_dabt>
ffff0014:	ea000481 	b	ffff1220 <vector_addrexcptn>
ffff0018:	ea000400 	b	ffff1020 <vector_irq>
ffff001c:	ea000487 	b	ffff1240 <vector_fiq>

...

ffff1020 <vector_irq>:
ffff1020:	e24ee004 	sub	lr, lr, #4
ffff1024:	e88d4001 	stm	sp, {r0, lr}
ffff1028:	e14fe000 	mrs	lr, SPSR
ffff102c:	e58de008 	str	lr, [sp, #8]
ffff1030:	e10f0000 	mrs	r0, CPSR
ffff1034:	e2200001 	eor	r0, r0, #1
ffff1038:	e16ff000 	msr	SPSR_fsxc, r0
ffff103c:	e20ee00f 	and	lr, lr, #15
ffff1040:	e1a0000d 	mov	r0, sp
ffff1044:	e79fe10e 	ldr	lr, [pc, lr, lsl #2]
ffff1048:	e1b0f00e 	movs	pc, lr
ffff104c:	80101cc0 	.word	0x80101cc0
ffff1050:	801018a0 	.word	0x801018a0

```
ffff1030:	e10f0000 	mrs	r0, CPSR
	arm 동작모드 점검 후
	이후 각 모드별로 `__irq_svc, __irq_usr` 브랜치

`__irq_svc`레이블
```
80101960 <__irq_svc>:
80101960:	e24dd04c 	sub	sp, sp, #76	; 0x4c
80101964:	e31d0004 	tst	sp, #4
80101968:	024dd004 	subeq	sp, sp, #4
8010196c:	e88d1ffe 	stm	sp, {r1, r2, r3, r4, r5, r6, r7, r8, r9, sl, fp, ip}
80101970:	e8900038 	ldm	r0, {r3, r4, r5}
80101974:	e28d7030 	add	r7, sp, #48	; 0x30
80101978:	e3e06000 	mvn	r6, #0
8010197c:	e28d204c 	add	r2, sp, #76	; 0x4c
80101980:	02822004 	addeq	r2, r2, #4
80101984:	e52d3004 	push	{r3}		; (str r3, [sp, #-4]!)
80101988:	e1a0300e 	mov	r3, lr
8010198c:	e887007c 	stm	r7, {r2, r3, r4, r5, r6}
80101990:	e1a096ad 	lsr	r9, sp, #13
80101994:	e1a09689 	lsl	r9, r9, #13
80101998:	e5991008 	ldr	r1, [r9, #8]
8010199c:	e3a0247f 	mov	r2, #2130706432	; 0x7f000000
801019a0:	e5892008 	str	r2, [r9, #8]
801019a4:	e58d104c 	str	r1, [sp, #76]	; 0x4c
801019a8:	eb040c41 	bl	80204ab4 <trace_hardirqs_off>
801019ac:	e59f1024 	ldr	r1, [pc, #36]	; 801019d8 <__irq_svc+0x78>
801019b0:	e1a0000d 	mov	r0, sp
801019b4:	e28fe000 	add	lr, pc, #0
801019b8:	e591f000 	ldr	pc, [r1]
801019bc:	eb040b8f 	bl	80204800 <trace_hardirqs_on>
801019c0:	e59d104c 	ldr	r1, [sp, #76]	; 0x4c
801019c4:	e5891008 	str	r1, [r9, #8]
801019c8:	e16ff005 	msr	SPSR_fsxc, r5
801019cc:	e24d0004 	sub	r0, sp, #4
801019d0:	e1801f92 	strex	r1, r2, [r0]
801019d4:	e8ddffff 	ldm	sp, {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, sl, fp, ip, sp, lr, pc}^
801019d8:	80ae9210 	.word	0x80ae9210

```
`80101960:	e24dd04c 	sub	sp, sp, #76	; 0x4c
	0x4c byte만큼 스텍공간 확보
`8010196c:	e88d1ffe 	stm	sp, {r1, r2, r3, r4, r5, r6, r7, r8, r9, sl, fp, ip}
	스텍공간에 r1부터 ip까지 저장
`801019ac:	e59f1024 	ldr	r1, [pc, #36]	; 801019d8 <__irq_svc+0x78>
	801019d8에 있는 0x80ae9210 를 r1에 로드
	0x80ae9210 에는 handle_arch_irq 전역변수 존재 >>System.map 에서 확인
	handle_arch_irq 전역변수에 저장된 함수 주소로 이동
		bcm2836_arm_irq_handle_irq()
		부팅시 bcm_arm_irqchip_l1_intc_of_init() 함수 구동 중 변수에 주소 할당
	즉 handle_arch_irq 전역변수는 인터럽트 컨트롤러 함수를 저장하는 전역변수

이후의 인터럽트 헨들러 호출까지 call stack
	bcm2836_arm_irq_handle_irq()
		`__handle_domain_irq()`
			generic_handle_irq()
				bcm2836_chained_handle_irq()
					generic_handle_irq()
						handle_fasteoi_irq()
							handle_irq_event()
								`__handle_irq_event_percpu()` 에서 헨들러 호출

irq 서브시스템 구성 함수에서 다양한 예외처리 수행

`__handle_domain_irq()`구현부
%%%%
```
int __handle_domain_irq(struct irq_domain *domain, unsigned int hwirq,
			bool lookup, struct pt_regs *regs)
{
	struct pt_regs *old_regs = set_irq_regs(regs);
	unsigned int irq = hwirq;
	int ret = 0;

	irq_enter();

#ifdef CONFIG_IRQ_DOMAIN
	if (lookup)
		irq = irq_find_mapping(domain, hwirq);
#endif

	/*
	 * Some hardware gives randomly wrong interrupts.  Rather
	 * than crashing, do something sensible.
	 */
	if (unlikely(!irq || irq >= nr_irqs)) {
		ack_bad_irq(irq);
		ret = -EINVAL;
	} else {
		generic_handle_irq(irq);
	}

	irq_exit();
	set_irq_regs(old_regs);
	return ret;
}
#endif
```

irq_enter() 내부에서 `__irq_enter` 호출


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



generic_handle_irq() 구현부
%%kernel/irq/irqdesc.c%%
```
int generic_handle_irq(unsigned int irq)
{
	struct irq_desc *desc = irq_to_desc(irq);

	if (!desc)
		return -EINVAL;
	generic_handle_irq_desc(desc);
	return 0;
}
```
irq_to_desc()를 통해 인터럽트 디스크립터 주소를 반환 [[irq_desc]]
	인터럽트 디스크립터는 request_riq() 에서 등록한 인터럽트 정보가 저장됨
		[[raspbian_4.19/interrupt/information|information]]


handle_fasteoi_irq() 구현부
%%kernel/irq/chip.c%%
```
void handle_fasteoi_irq(struct irq_desc *desc)
{
	struct irq_chip *chip = desc->irq_data.chip;

	raw_spin_lock(&desc->lock);
...
	kstat_incr_irqs_this_cpu(desc);
...
	handle_irq_event(desc);
...
```
kstat_incr_irqs_this_cpu(desc)
	인터럽트 발생횟수 저장

kstat_incr_irqs_this_cpu() 구현부
%%%%
```
static inline void __kstat_incr_irqs_this_cpu(struct irq_desc *desc)
{
	__this_cpu_inc(*desc->kstat_irqs);
	__this_cpu_inc(kstat.irqs_sum);
}
```
`__this_cpu_inc()`
	percpu타입 정수형 변수를 1증가


이후 handle_irq_event(desc); 에서`__handle_irq_event_percpu()` 호출


`__handle_irq_event_percpu()`구현부
%%kernel/irq/handle.c%%
```
irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc, unsigned int *flags)
{
	irqreturn_t retval = IRQ_NONE;
	unsigned int irq = desc->irq_data.irq;
	struct irqaction *action;

	record_irq_time(desc);

	for_each_action_of_desc(desc, action) {
		irqreturn_t res;

		trace_irq_handler_entry(irq, action);
		res = action->handler(irq, action->dev_id);
		trace_irq_handler_exit(irq, action, res);
...
```
res = action->handler(irq, action->dev_id);
	에서 핸들러 호출, irq는 인터럽트 번호, action->dev_id은 핸들러 전달 매개변수
	action->dev_id 의 경우 request_irq()로 헨들러 등록시 다섯번째 매개변수에 해당한다
		request_irq() 은 [[raspbian_4.19/interrupt/information|information]]에서 기술
trace_irq_handler_entry(irq, action); trace_irq_handler_exit(irq, action, res);
	에서 ftrace 이벤트 로그 출력