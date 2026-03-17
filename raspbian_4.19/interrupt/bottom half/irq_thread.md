부팅시 스레드 생성 과정 요약
	전달한 irq 스레드 정보를 해당 인터럽트 디스크립터에 설정
	kthread_create() 함수를 호출하여 irq 스레드 생성
		[[커널 스레드 생성]]

request_threaded_irq()
	`__setup_irq()`
		setup_irq_thread()
			kthread_create()
				>> irq 스레드 헨들러 함수, 프로세스 이름 : irq/%d-%s 지정


'irq 스레드 처리함수' = irq 스레드의 후반부를 처리하는 실질적 함수
irq 스레드 핸들러 함수 = irq_thread() 으로 'irq 스레드 처리함수'를 호출하는 역할

request_threaded_irq() 함수의 경우 인터럽트 설정 시 request_irq() 함수에서 호출된다
	[[raspbian_4.19/interrupt/information|information]]
	다른 점은 3번째 파라미터에 NULL or 스레드 헨들러 함수를 지정하냐 차이

선언부
%%include/linux/interrupt.h%%
```
request_threaded_irq(unsigned int irq, irq_handler_t handler,
		     irq_handler_t thread_fn,
		     unsigned long flags, const char *name, void *dev);

static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```
irq 스레드 처리 함수를 지정하면 커널 내부에서 irq 스레드를 생성한다

구현부
%%kernel/irq/manage.c%%
```
int request_threaded_irq(unsigned int irq, irq_handler_t handler,
			 irq_handler_t thread_fn, unsigned long irqflags,
			 const char *devname, void *dev_id)
{
	struct irqaction *action;
	struct irq_desc *desc;
	int retval;
...
action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
	if (!action)
		return -ENOMEM;

	action->handler = handler;
	action->thread_fn = thread_fn;
	action->flags = irqflags;
	action->name = devname;
	action->dev_id = dev_id;

	retval = irq_chip_pm_get(&desc->irq_data);
	if (retval < 0) {
		kfree(action);
		return retval;
	}

	retval = __setup_irq(irq, desc, action);
...
```
action에 irqaction 만큼 동적 메모리를 할당
인터럽트 디스크립터의 action필드에 값 저장 및 할당
irq 스레드가 등록되었거나, nested 변수가 0이면 setup_irq_thread() 함수 호출하여 스레드 생성

setup_irq_thread() 구현부
%%kernel/irq/manage.c%%
```
static int
setup_irq_thread(struct irqaction *new, unsigned int irq, bool secondary)
{
	struct task_struct *t;
	struct sched_param param = {
		.sched_priority = MAX_USER_RT_PRIO/2,
	};

	if (!secondary) {
		t = kthread_create(irq_thread, new, "irq/%d-%s", irq,
				   new->name);
	} else {
		t = kthread_create(irq_thread, new, "irq/%d-s-%s", irq,
				   new->name);
		param.sched_priority -= 1;
	}
...
```
kthread_create() 함수를 호출하여 스레드를 생성
	인자를 통해 스레드 이름을 확인할 수 있다
스레드 헨들러 함수 = irq_thread

핸들러가 실행되면 irq_thread()에서 무한루프를 돌며 목적에 맞는 동작 수행

%%kernel/irq/manage.c%%
```
static int irq_thread(void *data)
{
	struct callback_head on_exit_work;
	struct irqaction *action = data;
```
data로 irqaction 을 가져와 세부 속성 정보를 처리
제대로 된 irq_thread()구현부는 아래에 존재재

irq 스레드 생성 예제
---

%%drivers/mmc/host/bcm2835-mmc.c%%
```
static int bcm2835_mmc_add_host(struct bcm2835_host *host)
{
	struct mmc_host *mmc = host->mmc;
	struct device *dev = mmc->parent;
...
	bcm2835_mmc_init(host, 0);
	ret = request_threaded_irq(host->irq, bcm2835_mmc_irq,
				   bcm2835_mmc_thread_irq, IRQF_SHARED,
				   mmc_hostname(mmc), host);
...
```
86번 인터럽트 핸들러와 스레드를 설정하는 코드
	책과 다른점은 devm_~~ 코드가 작성되어있다
		자세한 사항은 1권 404p

인터럽트 핸들러로 bcm2835_mmc_irq() 를 지정
스레드 처리함수로 bcm2835_mmc_thread_irq()를 지정

다른 리눅스 시스템의 irq 스레드 생성 예제는 책 1권 407p



irq thread 실행
---
1. 인터럽트 핸들러에서  IRQ_WAKE_THREAD 반환
2. IRQ 스레드 깨움
3. IRQ 스레드 핸들러인 irq_thread()함수 실행
4. irq_thread()함수에서 irq 스레드 처리 함수 실행
[[interrupt 시 컨텍스트 흐름]]
`__handle_irq_event_percpu`함수에서 핸들러 실행
	 `__irq_wake_thread() 호출되면 깨움`

`__handle_irq_event_percpu`구현부
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

		if (WARN_ONCE(!irqs_disabled(),"irq %u handler %pF enabled interrupts\n",
			      irq, action->handler))
			local_irq_disable();

		switch (res) {
		case IRQ_WAKE_THREAD:

			if (unlikely(!action->thread_fn)) {
				warn_no_thread(irq, action);
				break;
			}

			__irq_wake_thread(desc, action);
```
action->handler 에서 핸들러 호출
이후 핸들러 반환값 res을 통해 switch문 실행, IRQ_WAKE_THREAD 인경우  `__irq_wake_thread(desc, action);`에서 깨움

IRQ_WAKE_THREAD는 bcm2835_mmc_irq() 함수에서 return 되는 것을 관찰 가능
%%%%
```
static irqreturn_t bcm2835_mmc_irq(int irq, void *dev_id)
{
	irqreturn_t result = IRQ_NONE;
	struct bcm2835_host *host = dev_id;
	u32 intmask, mask, unexpected = 0;
	int max_loops = 16;

	spin_lock(&host->lock);

	intmask = bcm2835_mmc_readl(host, SDHCI_INT_STATUS);
...
	do {
		/* Clear selected interrupts. */
		mask = intmask & (SDHCI_INT_CMD_MASK | SDHCI_INT_DATA_MASK |
				  SDHCI_INT_BUS_POWER);
		bcm2835_mmc_writel(host, mask, SDHCI_INT_STATUS, 8);

...
		if (intmask & SDHCI_INT_CARD_INT) {
			bcm2835_mmc_enable_sdio_irq_nolock(host, false);
			host->thread_isr |= SDHCI_INT_CARD_INT;
			result = IRQ_WAKE_THREAD;
		}
...
out:
	spin_unlock(&host->lock);

	if (unexpected) {
		pr_err("%s: Unexpected interrupt 0x%08x.\n",
			   mmc_hostname(host->mmc), unexpected);
		bcm2835_mmc_dumpregs(host);
	}

	return result;
}
```
result에 저장하고 이후 return


`__irq_wake_thread()`구현부
%%kernel/irq/handle.c%%
```
void __irq_wake_thread(struct irq_desc *desc, struct irqaction *action)
{
...
	atomic_inc(&desc->threads_active);

	wake_up_process(action->thread);
}
```
wake_up_process()함수를 호출하여 지정된 프로세스를 깨운다
action->thread 는 테스크 디스크립터가 저장되어 있다.


--------------------------------------------------------------------------
irq_thread() 함수 구현부
%%kernel/irq/manage.c%%
```
static int irq_thread(void *data)
{
	struct callback_head on_exit_work;
	struct irqaction *action = data;
	struct irq_desc *desc = irq_to_desc(action->irq);
	irqreturn_t (*handler_fn)(struct irq_desc *desc,
			struct irqaction *action);

	if (force_irqthreads && test_bit(IRQTF_FORCED_THREAD,
					&action->thread_flags))
		handler_fn = irq_forced_thread_fn;
	else
		handler_fn = irq_thread_fn;

	init_task_work(&on_exit_work, irq_thread_dtor);
	task_work_add(current, &on_exit_work, false);

	irq_thread_check_affinity(desc, action);

	while (!irq_wait_for_interrupt(action)) {
		irqreturn_t action_ret;

		irq_thread_check_affinity(desc, action);

		action_ret = handler_fn(desc, action);
		if (action_ret == IRQ_WAKE_THREAD)
			irq_wake_secondary(desc, action);

		wake_threads_waitq(desc);
	}
	task_work_cancel(current, irq_thread_dtor);
	return 0;
}
```
data를 통해 irqaction 포인터 action을 읽어온다(스레드 처리함수 thread_fn포함)
handler_fn = irq_thread_fn;
action_ret = handler_fn(desc, action);
	irq_thread_fn는 스레드 처리함수를 호출하는 함수로
	해당 함수를 바로 호출

%%kernel/irq/manage.c%%
```
static irqreturn_t irq_thread_fn(struct irq_desc *desc,
		struct irqaction *action)
{
	irqreturn_t ret;

	ret = action->thread_fn(action->irq, action->dev_id);
	if (ret == IRQ_HANDLED)
		atomic_inc(&desc->threads_handled);

	irq_finalize_oneshot(desc, action);
	return ret;
}
```

action->thread_fn으로 irq 스레드 처리 함수를 호출

인터럽트 핸들러에서 IRQ_WAKE_THREAD 반환되고, irq_wake_process로 깨워짐
스케쥴링 이후 흐름
kthread
	irq_thread
		irq_thread_fn
			bcm2835_mmc_thread_irq


실습 디버깅 과정은 나중에 추가 예정 > 1권 443p