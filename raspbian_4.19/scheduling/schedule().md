schedule_debug() 구현부
%%kernel/sched/core.c%%
```
static inline void schedule_debug(struct task_struct *prev)
{
#ifdef CONFIG_SCHED_STACK_END_CHECK
	if (task_stack_end_corrupted(prev))
		panic("corrupted stack end detected inside scheduler\n");
#endif

	if (unlikely(in_atomic_preempt_off())) {
		__schedule_bug(prev);
		preempt_count_set(PREEMPT_DISABLED);
	}
	rcu_sleep_check();

	profile_hit(SCHED_PROFILING, __builtin_return_address(0));

	schedstat_inc(this_rq()->sched_count);
}
```
인터럽트 컨텍스트에서 실행시 unlikely 부분 true 반환됨

`__schedule_bug()`구현부
%%kernel/sched/core.c%%
```
static noinline void __schedule_bug(struct task_struct *prev)
{
	/* Save this before calling printk(), since that will clobber it */
	unsigned long preempt_disable_ip = get_preempt_disable_ip(current);

	if (oops_in_progress)
		return;

	printk(KERN_ERR "BUG: scheduling while atomic: %s/%d/0x%08x\n",
		prev->comm, prev->pid, preempt_count());
...
```
많은 soc 벤더 들은 printk 이후 BUG()를 호출하여 강제 커널 패닉 유발