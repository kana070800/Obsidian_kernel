구현부
%%linux/include/linux/kthread.h%%
```
#define kthread_create(threadfn, data, namefmt, arg...) \
	kthread_create_on_node(threadfn, data, NUMA_NO_NODE, namefmt, ##arg)
	
struct task_struct *kthread_create_on_node(int (*threadfn)(void *data),
					   void *data,
					   int node,
					   const char namefmt[], ...);
```
인자
	`int (*threadfn)(void *data)`
		스레드 헨들러 함수 주소 저장(스레드 세부 동작 구현)
	`void *data`
		스레드 헨들러 함수에 전달하는 매개 변수
	int node
		노드 정보
	const char namefmt[]
		커널 스레드 이름 저장

[[kthread_create_on_node()]]

ex) 1권 > 176p