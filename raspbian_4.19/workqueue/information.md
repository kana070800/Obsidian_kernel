인터럽트나 프로세스 컨텍스트의 후반부 처리를 위해, 지정된 워크를 생성된 워커스레드에 할당하여 처리하는 기법 >> 프로세스 컨택스트이기에 휴면 가능, 스케줄링 가능
irq_thread는 특정 인터럽트 전용 스레드이고
work queue 는 공용작업 사무실
	`kworker/*`  형식으로 이름이 붙는 백그라운드 프로세스

일정 시간 이후 실행을 유도하는 딜레이 워크 기법 존재, 모니터링 용도

1. 워크를 큐잉
	schedule_work()
	insert_work()
2. 워커스레드를 깨움
	wake_up_worker()
3. 워커 스레드 실행
	process_one_work()

해당 버전의 경우 인터럽트 핸덜러 bcm2835_mbox_irq()에서 워크 큐잉
 > 스레드 핸들러 worker_thread()
 > 워크 핸들러 bcm2835_mbox_work_callback()

worker pool
---
워크큐를 관리하는 자료구조
	큐잉한 워크 리스트를 관리
	워커 스레드를 생성하고 관리

워크 풀 자료구조 >> worker_pool
%%kernel/workqueue.c%%
```
struct worker_pool {
	spinlock_t		lock;		/* the pool lock */
	int			cpu;		/* I: the associated cpu */
...
	struct list_head	worklist;	/* L: list of pending works */

	int			nr_workers;	/* L: total number of workers */
...
	struct list_head	workers;	/* A: attached workers */
...
```
worklist로 워크 리스트를 관리
workers로 워커 스레드를 관리

워크큐를 제어하는 자료구조들의 구조도, 설명은 아래 링크
[[워크큐 자료구조 overall]]

