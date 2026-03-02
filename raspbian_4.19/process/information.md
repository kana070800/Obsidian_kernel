systemd
	라즈비안에서 모든 프로세스를 관리하는 init 프로세스
	대부분의 리죽스 배포판에서는 pid=1 인 프로세스를 init으로 부른다
	모든 유저 프로세스의 부모(init)
kthreadd
	모든 커널 스레드의 부모

프로세스 생성 [[_do_fork()]]
	`_do_fork()` 함수를 호출한다는 공통점
		유저 프로세스
			라이브러리의 도움을 받아 커널에게 요청(gnu c : gilbc)
			fork() 나 pthread_create() 함수 호출
				sys_clone 을 통해 `_do_fork()`호출-- [[syscalls]] 
		커널 프로세스
			kthread_create() 호출하여 생성
				내부에서 `_do_fork()`호출


생성 과정 call_stack (in usr mode)
	copy_process.part.*
	`_do_fork`
	sys_clone
	ret_fast_syscall

종료 과정 call_stack (in usr mode)   from kill signal (9)
	do_exit
	do_group_exit
	get_signal
	do_signal
	do_work_pending
	slow_work_pending
	종료 시 시그널 17번 생성 > 부모에 전달

종료 과정 call_stack (in usr mode)   from exit system call
	do_exit
	do_group_exit
	`__wake_up_parent+0x0` >> sys_group_exit() 이 실제로 호출------ `1권 277p`
	ret_fast_syscall

whoami, cat 시스템 콜 실습
---
생성 과정 call_stack
sched_process_fork ftrace log 
execve 시스템 콜 발생
	search_binary_handler
	`__do_execve_file`
	do_execve
	sys_execve
	ret_fast_syscall

파일 시스템의 파일이 메모리에 적재되어 프로세스가 되었다

sys_execve() > do_execveat_common() > exec_binprm() > trace_sched_process_exec 실행
whoami 프로세스 종료 call_stack

리눅스 유틸리티 프로그램을 실행할 때 fork와 execve 시스템 콜을 호출한다
종료시 exit 시스템 콜을 호출한다
sched_process_exec 이벤트 ftrace로 파일 위치를 알 수 있다

