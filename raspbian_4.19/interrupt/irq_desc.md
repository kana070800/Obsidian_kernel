인터럽트 디스크립터

인터럽트 번호를 입력으로 irq_to_desc() 함수를 통해 인터럽트 디스크립터 주소를 알 수 있다

시스템에 종류별로 지정된 인터럽트 핸들러 호출을 위해 접근해야한다

request_irq()함수를 통해 인터럽트 디스크립터에 핸들러 등을 등록하고 필요시 호출

%%include/linux/irqdesc.h%%
```
struct irq_desc {
	struct irq_common_data	irq_common_data;
	struct irq_data		irq_data;
	unsigned int __percpu	*kstat_irqs;
	irq_flow_handler_t	handle_irq;
#ifdef CONFIG_IRQ_PREFLOW_FASTEOI
	irq_preflow_handler_t	preflow_handler;
#endif
	struct irqaction	*action;	/* IRQ action list */
	unsigned int		status_use_accessors;
...
```
필드들
	struct irq_common_data	irq_common_data
		irq_chip 관련함수정보
	struct irq_data		irq_data
		인터럽트 번호와 하드웨어 핀번호
	`unsigned int __percpu	*kstat_irqs`
		인터럽트 발생횟수
	`struct irqaction	*action`
		인터럽트 세부 핵심 속성
			

irqaction 선언부
%%include/linux/interrupt.h%%
```
struct irqaction {
	irq_handler_t		handler;
	void			*dev_id;
	void __percpu		*percpu_dev_id;
	struct irqaction	*next;
	irq_handler_t		thread_fn;
	struct task_struct	*thread;
	struct irqaction	*secondary;
	unsigned int		irq;
	unsigned int		flags;
	unsigned long		thread_flags;
	unsigned long		thread_mask;
	const char		*name;
	struct proc_dir_entry	*dir;
} ____cacheline_internodealigned_in_smp;
```
필드들
	irq_handler_t		handler
		인터럽트 핸들러 주소
	`void			*dev_id`
		인터럽트 핸들러에 전달되는 매개변수(드라이버 핸들링 구조체 주소가 주됨)\
	irq_handler_t		thread_fn
		irq 스레드 방식으로 처리할 때 irq스레드 처리함수의 주소를 저장
	unsigned int		irq
		인터럽트 번호
	unsigned int		flags
		인터럽트 설정 플래그 필드 (IRQF_TRIGGER_* 형식으로 정의됨)


