
아키텍처 종속적(cpu마다 구현방식, 구조가 다름)

프로세스의 실행흐름관리(스레드 정보)
	컨텍스트 정보(인터럽트 컨텍스트인지 여부, 선점 가능 조건인지 여부)
	스케줄링 직전 실행했던 레지스터 세트 저장, 로딩
	프로세스 세부 실행 정보(시그널 받았는지 여부)

각 프로세스 스택 공간 최상단 주소에 위치
	arm 아키텍처에서 프로세스 스택 크기는 0x2000바이트

선언부 위치
%%arch/arm/include/asm/thread_info.h%%
```
struct thread_info {
	unsigned long	    	flags;		/* low level flags */
	int		                preempt_count;	/* 0 => preemptable, <0 => bug */
	mm_segment_t	    	addr_limit;	/* address limit */
	struct task_struct   	*task;		/* main task structure */
	__u32			        cpu;		/* cpu */
	__u32		    	    cpu_domain;	/* cpu domain */
	struct cpu_context_save	cpu_context;	/* cpu context */
	__u32			        syscall;	/* syscall number */
	__u8			        used_cp[16];	/* thread used copro */
	unsigned long	    	tp_value[2];	/* TLS registers */
#ifdef CONFIG_CRUNCH
	struct crunch_state	    crunchstate;
#endif
	union fp_state		    fpstate __attribute__((aligned(8)));
	union vfp_state	    	vfpstate;
#ifdef CONFIG_ARM_THUMBEE
	unsigned long		    thumbee_state;	/* ThumbEE Handler Base register */
#endif
};
```

주요필드
	unsigned long flags
		flags 필드를 수시로 체크하며 시크널, 선점, 시스템콜 트레이싱 조건을 점검한다
		%%/arch/arm/include/asm/thread_info.h 에 지정된 flags%%
			`#define _TIF_SIGPENDING	(1 << TIF_SIGPENDING)`  >> 시그널 전달됨
			`#define _TIF_NEED_RESCHED	(1 << TIF_NEED_RESCHED)` >>> 선점될 조건
			`#define _TIF_NOTIFY_RESUME	(1 << TIF_NOTIFY_RESUME)`
			`#define _TIF_SYSCALL_TRACE	(1 << TIF_SYSCALL_TRACE)`
	int preempt_count
		프로세스의 컨텍스트 정보를 저장
			irq 컨텍스트, soft_riq 컨텍스트, 선점될 조건 등 저장
		[[interrupt 시 컨텍스트 흐름]]
		[[softirq에서 컨텍스트 흐름]]
	`__u32 cpu`
		실행중인 cpu 번호 >> raw_smp_process_id() 로 알 수 있다
	`struct task_struct *task`
		[[task_struct]] 주소 저장
	struct cpu_context_save cpu_context
		스케줄링 전후로 cpu  레지스터 세트 정보 저장
		컨텍스트 스위칭 시 저장 및 로드 >> switch_to() > `__switch_to` 에서 구현

함수 호출시 발생하는 푸시, 팝의 동작의 경우 어셈블리 코드에서 확인가능

%%arch/arm/include/asm/thread_info.h%%
```
struct cpu_context_save {
	__u32	r4;
	__u32	r5;
	__u32	r6;
	__u32	r7;
	__u32	r8;
	__u32	r9;
	__u32	sl;
	__u32	fp;
	__u32	sp;
	__u32	pc;
	__u32	extra[2];		/* Xscale 'acc' register, etc */
};
```

인터럽트의 경우 실행중인 프로세스의 레지스터 세트를 프로세스 스텍에 저장한다
preempt_count()
---
%%include/asm-generic/preempt.h%%
```
static __always_inline int preempt_count(void)
{
	return READ_ONCE(current_thread_info()->preempt_count);
}
```
스택 최상단 주소의 thread_info 의 preempt_count 필드를 얻어온다
%%arch/arm/include/asm/thread_info.h%%
```
register unsigned long current_stack_pointer asm ("sp");

static inline struct thread_info *current_thread_info(void)
{
	return (struct thread_info *)
		(current_stack_pointer & ~(THREAD_SIZE - 1));
}
```
register를 사용하여 스택주소를 current_stack_pointer 변수로 읽는 동작
현재 실행중인 프로세스의 스택 최상단 주소 계산
인라인 어셈블리 asm("sp") 사용
cpu 필드 분석
---
 raw_smp_process_id() 구현부
 %%include/linux/smp.h%%
```
 # define smp_processor_id() raw_smp_processor_id()
```
%%arch/arm/include/asm/smp.h%%
```
#define raw_smp_processor_id() (current_thread_info()->cpu)
```

cpu필드는 set_task_cpu() 호출 시 변경
 %%kernel/sched/core.c%%
```
void set_task_cpu(struct task_struct *p, unsigned int new_cpu)
{
...
	__set_task_cpu(p, new_cpu);
}
```
%%kernel/sched/sched.h%%
```
static inline void __set_task_cpu(struct task_struct *p, unsigned int cpu)
{
...
	WRITE_ONCE(task_thread_info(p)->cpu, cpu);
}
```
task_thread_info() 는 task_struct를 입력으로 프로세스 스택 최상단 주소 반환> cpu 값전달
task_struct 보다 thread_info가 프로세스 세부 동작을 저장하는 역할을 수행한다


thread_info 초기화
---
프로세스 생성 시 thread_info 구조체 초기화 과정
copy_process()내부에서 호출한 dup_task_struct() 내부
	alloc_task_struct_node() > task_struct 구조체 할당
	alloc_thread_stack_node() > 프로세스 스택 공간 할당
	[[_do_fork()]] 에서 분석