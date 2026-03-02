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
	유저모드에서 sys_clone 시스템 콜 헨들러(유저 프로세스 생성 시 시스템 콜 관여)
	커널 모드에서 kernel_thread() 함수

전체적인 동작
	1. 프로세스 생성
		copy_process() 호출
	2. 생성한 프로세스 실행요청
		wake_up_new_task() 호출

%%linux/kernel/fork.c%%
```
long _do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr,
	      unsigned long tls)
{
	struct completion vfork;
	struct pid *pid;
	struct task_struct *p;
	int trace = 0;
	long nr;
...
	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
	add_latent_entropy();

	if (IS_ERR(p))
		return PTR_ERR(p);

	trace_sched_process_fork(current, p);

	pid = get_task_pid(p, PIDTYPE_PID);
	nr = pid_vnr(pid);

...
	wake_up_new_task(p);

...

	put_pid(pid);
	return nr;
}
```
copy_process() 호출하여 프로세스 복사
p 포인터 오류 점검
ftrace events log(sched_process_fork)
프로세스 pid 계산
생성한 프로세스를 깨우기> wake_up_new_task(p)
프로세스 pid 반환

copy_process()
%%/linux/kernel/fork.c%%
```
static __latent_entropy struct task_struct *copy_process(
					unsigned long clone_flags,
					unsigned long stack_start,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace,
					unsigned long tls,
					int node)
{
	int retval;
	struct task_struct *p;
	struct multiprocess_signals delayed;
...
	retval = -ENOMEM;
	p = dup_task_struct(current, node);
	if (!p)
		goto fork_out;
...
	/* Perform scheduler related setup. Assign this task to a CPU. */
	retval = sched_fork(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_policy;
...
	retval = copy_files(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_semundo;
	retval = copy_fs(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_files;
	retval = copy_sighand(clone_flags, p);
	if (retval)
		goto bad_fork_cleanup_fs;
...
```
dup_task_struct() > task_struct 구조체 할당(stack)
sched_fork() > task_struct 스케쥴링 관련 정보 초기화
copy_* > 파일 디스크립터 관련 내용 초기화, 복사 등
copy_sighand() > 시그널 핸들러 정보(sighand_struct 구조체) 생성, 복사
>>> 새로 생성하는 프로세스 테스크 디스크립터 복제

dup_task_struct() 구현부
%%kernel/fork.c%%
```
static struct task_struct *dup_task_struct(struct task_struct *orig, int node)
{
	struct task_struct *tsk;
	unsigned long *stack;
	struct vm_struct *stack_vm_area;
	int err;
	
		if (node == NUMA_NO_NODE)
		node = tsk_fork_get_node(orig);
	tsk = alloc_task_struct_node(node);
	if (!tsk)
		return NULL;

	stack = alloc_thread_stack_node(tsk, node);
	if (!stack)
		goto free_tsk;
...
	tsk->stack = stack;
...
	setup_thread_stack(tsk, orig);
...
	return tsk;
```
alloc_task_struct_node() > task_struct 할당
alloc_thread_stack_node() > 스택 메모리 공간 할당 (0x2000)
tsk->stack = stack; > stack 필드에 스택 주소 저장(스택 최상단)
setup_thread_stack() > task_struct 주소를 thread_info.task 에 저장

alloc_task_struct_node() 구현부
%%kernel/fork.c%%
```
static struct kmem_cache *task_struct_cachep;

static inline struct task_struct *alloc_task_struct_node(int node)
{
	return kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
}
```
task_struct_cachep 슬랩 캐시를 통해 task_struct 구조체 할당
	자주 사용하는 객체를 만들어두고 필요시 할당([[슬랩 캐시]]에서 다룸)


alloc_thread_stack_node() 구현부
%%kernel/fork.c%%
```
static unsigned long *alloc_thread_stack_node(struct task_struct *tsk, int node)
{
#ifdef CONFIG_VMAP_STACK
...
#else
	struct page *page = alloc_pages_node(node, THREADINFO_GFP,
					     THREAD_SIZE_ORDER);

	if (likely(page)) {
		tsk->stack = page_address(page);
		return tsk->stack;
	}
	return NULL;
#endif
}

```
페이지를 할당하는 동작, THREAD_SIZE_ORDER == 1 이니 2개 페이지 할당 (0x2000)
page_address()를 호출해 페이지를 [[가상주소]]로 바꾼다음 반환

setup_thread_stack() 구현부
%%include/linux/sched/task_stack.h%%
```
static inline void setup_thread_stack(struct task_struct *p, struct task_struct *org)
{
	*task_thread_info(p) = *task_thread_info(org);
	task_thread_info(p)->task = p;
}
```
 `struct task_struct *org` > 부모 프로세스의 task_struct
  `struct task_struct *p` > 생성한 프로세스의 task_struct
  부모 프로세스 thread_info 구조체 필드를 복사
  thread_info 의 task 에 task_struct 주소 저장 


wake_up_new_task()
%%linux/kernel/sched/core.c%%
```
void wake_up_new_task(struct task_struct *p)
{
	struct rq_flags rf;
	struct rq *rq;

	raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
	p->state = TASK_RUNNING;
#ifdef CONFIG_SMP
	/*
	 * Fork balancing, do it here and not earlier because:
	 *  - cpus_allowed can change in the fork path
	 *  - any previously selected CPU might disappear through hotplug
	 *
	 * Use __set_task_cpu() to avoid calling sched_class::migrate_task_rq,
	 * as we're not fully set-up yet.
	 */
	p->recent_used_cpu = task_cpu(p);
	__set_task_cpu(p, select_task_rq(p, task_cpu(p), SD_BALANCE_FORK, 0));
#endif
	rq = __task_rq_lock(p, &rf);
	update_rq_clock(rq);
	post_init_entity_util_avg(&p->se);

	activate_task(rq, p, ENQUEUE_NOCLOCK);
	p->on_rq = TASK_ON_RQ_QUEUED;
	trace_sched_wakeup_new(p);
	check_preempt_curr(rq, p, WF_FORK);
```
프로세스 p->state 를 TASK_RUNNING 으로 변경
`__set_task_cpu()`를 호출하여 thread_info 구조체의 cpu필드에 현재 cpu번호 저장
`__task_rq_lock()`을 통해 런큐 주소를 읽고
activate_task() 를 통해 런큐에 생성한 프로세스 삽입

 >>> 이후 스케쥴러는 런큐를 점검, 우선순위를 계산해 실행 여부 결정