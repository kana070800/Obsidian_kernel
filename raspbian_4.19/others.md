gcc 컴파일러 제공, 전처리 메크로
	`__func__` >> 현재 실행 중인 함수이름
	`__LINE__` >> 현재 실행 중인 코드 라인
	`__builtin_return_address(0)` >> 현재 실행 중인 함수를 호출한 함수 주소

ftrace
---
	tracer
		nop >> 기본 트레이서, 이벤트만 출력
		function >> 함수 호출 스텍 출력
		fucntion_graph >> 실행 시간과 세부 호출 정보를 그래프로 출력
	events
		ftrace에서 제공하는 이벤트 종류 확인 가능
			irq, sched 등
	available_filter_functions
		추적가능한 함수 목록, 새로 구현한 함수 역시 가능
		inline함수의경우 추적 불가
	컨텍스트 정보
		d : 해당 cpu의 인터럽트 비활성화 상태
		n : 현재 프로세스가 선점 스케쥴링 가능
		h/s : 인터럽트 컨텍스트 / soft irq 컨텍스트
		0~3 : 프로세스 [[thread_info]] 구조체의 preempt_count 필드값
	커널 코드 내부 trace_* 함수가 바로 ftrace 출력함수
		trace_printk >> 원하는 출력값을 출력할 수 있는 함수

커널 입장에서 스레드는 프로세스와 동등하게 본다

`__noreturn` >> 실행 후 자신을 호출한 함수로 되돌아가지 않는다 ex) do_exit() > schedule()


info "명령어" >> 해당 명령어의 정보 출력

함수 호출시 발생하는 푸시, 팝의 동작의 경우 어셈블리 코드에서 확인가능

인터럽트 컨텍스트인지 확인하는 방법
	in_interrupt() 제공


리눅스 유틸리티 프로그램을 실행할 때 fork와 execve 시스템 콜을 호출한다 (whoami, cat)
종료시 exit 시스템 콜을 호출한다
sched_process_exec 이벤트 ftrace로 파일 위치를 알 수 있다


gdb를 통한 시스템콜 어셈블리 디버깅 방법
--
	책 1권 263p



current
---
current는 현재 실행중인 프로세스 테스크 디스크립터 자료구조 (task_struct)
테스크 디스크립터 주소를 반환하는 메크로 함수

%%include/asm-generic/current.h%%
```
#define get_current() (current_thread_info()->task)
#define current get_current()
```
current_thread_info() >> thread_info 구조체의 시작 주소 반환

current_thread_info() 구현부
%%arch/arm/include/asm/thread_info.h%%
```
register unsigned long current_stack_pointer asm ("sp");

static inline struct thread_info *current_thread_info(void)
{
	return (struct thread_info *)
		(current_stack_pointer & ~(THREAD_SIZE - 1));
}
```
THREAD_SIZE == 0x2000
register unsigned long current_stack_pointer asm ("sp")
	현재 프로세스의 스택 주소를 current_stack_pointer로 가져오라는 명령
	최상단 주소를 비트연산을 통해 연산
	자세한 계산 과정은 1권 257p