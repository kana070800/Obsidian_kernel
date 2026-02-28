systemd
	라즈비안에서 모든 프로세스를 관리하는 init 프로세스
	대부분의 리죽스 배포판에서는 pid=1 인 프로세스를 init으로 부른다
	모든 유저 프로세스의 부모(init)
kthreadd
	모든 커널 스레드의 부모

프로세스 생성
	`_do_fork()` 함수를 호출한다는 공통점 [[_do_fork()]]
		유저 프로세스
			fork() 나 pthread_create() 함수 호출
				sys_clone -- [[syscalls]]
			라이브러리의 도움을 받아 커널에게 요청(gnu c : gilbc)
		커널 프로세스
			kthread_create() 호출하여 생성


생성 과정 call_stack (in usr mode)
	copy_process.part.5
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
	`__wake_up_parent+0x0` >> sys_exit_group 이 실제로 호출 
	ret_fast_syscall

