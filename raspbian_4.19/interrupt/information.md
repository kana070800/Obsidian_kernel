
armv7에서 인터럽트는 exception의 한 종류로 처리

exception 발생시 exception모드로 전환
	정해진 주소로 branch (<vector_irq>)
	exception 벡터 테이블 베이스 주소에서 offset 주소만큼 더한 주소로 프로그램 카운터 이동
	해당 과정은 하드웨어적으로 처리
[[interrupt 시 컨텍스트 흐름]]

대부분의 드라이버는 인터럽트를 통해 하드웨어 디바이스 드라이버와 통신

인터럽트 컨텍스트에서 스케줄링 지원 함수 호출시 커널 패닉이나 WARN() 호출하여 에러발생
	[[schedule()]]
		schedule_debug()
			`__schedule_debug()`

/proc/interrupts 파일을 통해 인터럽트 디버깅 인터페이스를 확인
읽으면 커널 내부 show_interrupts() 함수가 실행되어 보여준다
%%kernel/irq/proc.c%%
```
int show_interrupts(struct seq_file *p, void *v)
{
	static int prec;

	unsigned long flags, any_count = 0;
	int i = *(loff_t *) v, j;
	struct irqaction *action;
	struct irq_desc *desc;
...
	rcu_read_lock();
	desc = irq_to_desc(i);
...
	seq_printf(p, "%*d: ", prec, i);
	for_each_online_cpu(j)
		seq_printf(p, "%10u ", desc->kstat_irqs ?
					*per_cpu_ptr(desc->kstat_irqs, j) : 0);
...
#ifdef CONFIG_GENERIC_IRQ_SHOW_LEVEL
	seq_printf(p, " %-8s", irqd_is_level_type(&desc->irq_data) ? "Level" : "Edge");
#endif
	if (desc->name)
		seq_printf(p, "-%-8s", desc->name);

	action = desc->action;
	if (action) {
		seq_printf(p, "  %s", action->name);
		while ((action = action->next) != NULL)
			seq_printf(p, ", %s", action->name);
	}
...
```
desc = irq_to_desc(i);
	인터럽트 디스크립터 주소를 읽기
for_each_online_cpu(j) ~~
	각 cpu 별로 인터럽트 번호, 횟수를 출력
if (action) ~~
	하나의 인터럽트가 여러이름으로 등록되었을 시 이름들을 출력
proc 가상파일 시스템관련 동작 >>>> 1권 371p 에 기술


인터럽트 비활성화가 필요한 상황
	soc에서 정의한 하드웨어블록에 정확한 시퀀스를 줘야할 경우
	유휴상태 진입 전 시스템 상태정보를 저장하는 동작
	디바이스 드라이버에 데이터 시트에 명시한 대로 정확한 특정 시퀀스를 줘야할 경우
	예외 발생하여 시스템 리셋 전
	
	local_irq_disabled(), local_irq_enabled()로 해당 cpu 인터럽트를 비활성화가능
	cpsid i 어셈블리 명령으로 해당 cpu 인터럽트를 비활성화가능



선점 스케줄링의 진입경로중 하나가 인터럽트 종료 시점
시그널 헨들러는 인터럽트 핸들러 종료 시점에 수행

인터럽트 절차
	인터럽트 벡터 실행
		레지스터 세트를 스택에 푸시 [thread_info가 아닐거다]
	커널 인터럽트 내부 함수 호출
	인터럽트 핸들러 호출 (irq context start)
		인터럽트 디스크립터 [[irq_desc]] 를 통해 호출

in_interrupt() 를 통해 인터럽트 컨텍스트 판별 가능




in_interrupt()
---
구현부
%%include/linux/preempt.h%%
```
#define in_interrupt()		(irq_count())
```
```
#define irq_count()	(preempt_count() & (HARDIRQ_MASK | SOFTIRQ_MASK \
				 | NMI_MASK))
```

 (HARDIRQ_MASK == 0xf0000,  SOFTIRQ_MASK == 0xff00, NMI_MASK == 0x100000)

preempt_count의 구현부는[[thread_info]]에 기재


인터럽트 발생횟수 확인
---
kstat_irqs_cpu() 함수는 인터럽트 종류별로 발생횟수를 알려주는 기능 수행

kstat_irqs_cpu()구현부
%%kernel/irq/irqdesc.c%%
```
unsigned int kstat_irqs_cpu(unsigned int irq, int cpu)
{
	struct irq_desc *desc = irq_to_desc(irq);

	return desc && desc->kstat_irqs ?
			*per_cpu_ptr(desc->kstat_irqs, cpu) : 0;
}
```
irq_to_desc(irq);
	인터럽트 디스크립터 주소 저장
return desc && ...
	인터럽트 처리횟수 반환
인터럽트 핸들러 등록과정
---
부팅과정에서 인터럽트 핸들러 등록

request_irq() 선언부
%%include/linux/interrupt.h%%
```
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
	    const char *name, void *dev)
{
	return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}
```
인자들
	unsigned int irq
		인터럽트 번호
	irq_handler_t handler
		핸들러 주소
	unsigned long flags
		인터럽트의 속성 플래그 >> 예제는 1권 351p
	`const char *name
		인터럽트 이름
	`void *dev
		인터럽트 핸들러의 매개변수(보통 드라이버 제어하는 구조체 주소)

request_threaded_irq() 구현부
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
	desc = irq_to_desc(irq);
	if (!desc)
		return -EINVAL;
...
	action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
	if (!action)
		return -ENOMEM;
		
	action->handler = handler;
	action->thread_fn = thread_fn;
	action->flags = irqflags;
	action->name = devname;
	action->dev_id = dev_id;
```
desc = irq_to_desc(irq);
	인터럽트 주소 가져오기
action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
	irqaction 구조체 크기만큼 동적 메모리 할당
이후 `action->* = *;`
	irqaction 의 각 구조체 필드에 request_irq 로 전달된 인자 저장

request_irq() 의 사용코드
%%drivers/usb/host/dwc_otg/dwc_otg_driver.c%%
```
static int dwc_otg_driver_probe(
#ifdef LM_INTERFACE
				       struct lm_device *_dev
...
	int retval = 0;
	dwc_otg_device_t *dwc_otg_device;
        int devirq;
...
	retval = request_irq(devirq, dwc_otg_common_irq,
                             IRQF_SHARED,
                             "dwc_otg", dwc_otg_device);
```

