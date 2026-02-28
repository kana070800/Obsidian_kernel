gcc 컴파일러 제공, 전처리 메크로
	`__func__` >> 현재 실행 중인 함수이름
	`__LINE__` >> 현재 실행 중인 코드 라인
	`__builtin_return_address(0)` >> 현재 실행 중인 함수를 호출한 함수 주소

ftrace
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

info "명령어" >> 해당 명령어의 정보 출력