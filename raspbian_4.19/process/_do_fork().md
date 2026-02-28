선언부
%%linux/kernel/fork.c%%
```
long _do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr,
	      unsigned long tls)
```
인자
	clone_flags
		프로세스 생성시 옵션 정보(복제될 리소스 정보)
		%%include/uapi/linux/sched.h 에 지정된 플레그들%%
	stack_start, stack_size
		복사하려는 스택의 주소, 실행중인 스택 크기
	`*parent_tidptr, *child_tidptr`
		부모와 자식 스레드 그룹관리 헨들러 정보

언제 호출하는가?
	유저모드에서 sys_clone 시스템 콜 헨들러
	커널 모드에서 kernel_thread() 함수