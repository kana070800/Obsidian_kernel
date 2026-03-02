테스크 디스크립터
pid_t == int == `__kernel_pid_t`

아키텍처 독립적

current는 현재 실행중인 프로세스 테스크 디스크립터 자료구조 (task_struct)

테스크 디스크립터의 각 필드가 어느 코드에서 바뀌게 되는지 파악해 보는 것도 좋은 방법
task_struct의 연결리스트 init_task.tasks가 존재하고 ps 명령시 여기서 읽어옴

선언부 위치%%include/linux/sched.h%%

주요필드
	`char comm[TASK_COMM_LEN]`
		프로세스 이름 저장
	pid_t pid
		pid값
	pid_t tgid
		스레드의 그룹 아이디
	`void * stack`
		[[thread_info]] 주소(스택 최상단 주소)
	files
		파일 디스크립터
	volatile long state
		프로세스 실행 상태 %% include/linux/sched.h 에 task_* %%
			`#define TASK_RUNNING			0x0000`  실행중이거나 대기
				`#define TASK_INTERRUPTIBLE		0x0001`  휴면 
			`#define TASK_UNINTERRUPTIBLE	0x0002`  특정조건에서 깨어가기 위한 휴면
			`#define __TASK_STOPPED			0x0004`
			`#define __TASK_TRACED			0x0008`
	unsigned int flags
		프로세스 세부 동작 상태와 속성 %% include/linux/sched.h 에 PF_* %%
			`#define PF_IDLE			0x00000002	/* I am an IDLE thread */
			`#define PF_EXITING		0x00000004	/* Getting shut down */
			`#define PF_EXITPIDONE		0x00000008	/* PI exit done on shut down */
			`#define PF_VCPU			0x00000010	/* I'm a virtual CPU */
			`#define PF_WQ_WORKER		0x00000020	/* I'm a workqueue worker */
		ex code >> 1권 207p
	int exit_state
		프로세스 종료상태 저장
			`#define EXIT_DEAD			0x0010
			`#define EXIT_ZOMBIE			0x0020
			`#define EXIT_TRACE			(EXIT_ZOMBIE | EXIT_DEAD)
	int exit_code
		프로세스 종료코드 저장(do_exit의 인자를 저장하여 종료 옵션 지정)
			tsk->exit_code =code; in do_exit function
	`struct task_struct *real_parent`
		자신을 생성한 부모 프로세스의 task_struct 주소
	`struct task_struct *parent`
		현재 부모 프로세스의 task_struct 주소(본래 부모 종료시 조부모가 부모로 취급)
	struct list_head children
		자식 프로세스 생성 시 children 연결 리스트에 등록
	struct list_head sibling
		같은 부모가 생성한 프로세스의 연결 리스트 주소(형제 관계)
	struct list_head	tasks
		구동중인 모든 프로세스가 등록된 연결 리스트
			copy_process() 함수 수행 도중 init_task.tasks 연결리스트에 등록
			%%linux/kernel/fork.c 내부
				list_add_tail_rcu(&p->tasks, &init_task.tasks)%%
		list_head의 next 필드는 연결리스트의 다음 list_head의 주소를 가리킨다
			1권 > 213p
		커널에서 구동중인 모든 태스크 디스크립터 주소를 알 수 있다.
	u64 utime
		유저모드에서 프로세스가 실행한 시각
			account_user_time() 함수에서 바뀜
				%%kernel/sched/cputime.c 내부
					p->utime += cputime;%%
	u64 stime
		커널모드에서 프로세스가 실행한 시각
			account_system_index_time() 에서 바뀜
				%%kernel/sched/cputime.c 내부
					p->stime += cputime;%%
	struct sched_info sched_info.last_arrival
		sched_info는 프로세스 스케줄링 정보 저장
		last_arrival는 마지막에 cpu에서 실행된 시간
			sched_info_arrive() 에서 바뀜
				%%kernel/sched/core.c 내부
					t->sched_info.last_arrival = now;%%
			context_switch() > prepare_task_switch() > ... > sched_info_arrive() 흐름으로 실행
			즉 context_switch 내에서 컨택스트 스위칭 직전 실행시간을 업데이트한다