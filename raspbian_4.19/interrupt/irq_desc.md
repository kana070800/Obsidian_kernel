인터럽트 디스크립터

시스템에 종류별로 지정된 인터럽트 핸들러 호출을 위해 접근해야한다

%%include/linux/irqdesc.h%%
```
struct irq_desc {
	struct irq_common_data	irq_common_data;
	struct irq_data	        irq_data;
	unsigned int __percpu	*kstat_irqs;
	irq_flow_handler_t	    handle_irq;
...
```