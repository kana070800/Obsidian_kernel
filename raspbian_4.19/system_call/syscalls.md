
SYSCALL_DEFINE 메크로로 함수이름 지정 시 빌드 과정에서 sys_* 형태로 심벌 생성

sys_getpid()
%%linux/kernel/sys.c%%
```
SYSCALL_DEFINE(getpid)
{
	return task_tgid_vnr(current);
}
```

sys_clone()
%%%linux/kernel/fork.c%%
```
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
		 int __user *, parent_tidptr,
		 unsigned long, tls,
		 int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
...
#endif
{
	return _do_fork()(clone_flags, newsp, 0, parent_tidptr, child_tidptr, tls);
}
#endif
```
[[_do_fork()]]
