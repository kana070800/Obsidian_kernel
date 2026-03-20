인터럽트 후반부 처리를 기술

[[irq_thread]]
	전용 커널 irq thread를 생성하여 인터럽트 후반부처리 실행
		스레드 이름의 경우 "irq/num-name" 형식으로 구성됨
	인터럽트 핸들러에서 IRQ_THREAD_WAKE 반환시 깨워짐

[[softirq]]
	인터럽트 핸들러 직후 즉시 실행, 빠르게 처리하고 싶다면 실행
	별도의 softriq 컨택스트로 판정
	네트워크 고속 패킷이나 스토리지 디바이스에서 사용
	softirq가 오래 걸리면 ksoftirqd 프로세스가 나머지 실행
	percpu 인 softirq_vec에서 호출

[[tasklet]]
	동적으로 softirq를 쓸 수 있게 한 인터페이스
	percpu 인 tasklet_vec에서 호출

workqueue
	인터럽트 핸들러 실행시 워크를 워크큐에 큐잉하고 프로세스 레벨의 워커 스레드에서 후반부 처리

irq 스레드의 경우 RT 프로세스로 구동되어 선점 스케쥴링 불가
	자주 발생하는 인터럽트에 부적합(워크큐도 마찬가지)
자주발생하는 경우 softirq나 tasklet이 적합


softirq 컨택스트 확인
---
in_softirq() 함수를 사용
%%include/linux/preempt.h%%
```
#define in_softirq()		(softirq_count())
```



디버깅 과정은 나중에 추가 예정>>  책 1권 528p


softirq 처리 횟수는 show_softirqs 함수에서 출력
횟수를 업데이트는 `__do_softirq`에서 kstat_incr_softirqs_this_cpu() 함수를 호출한 시점
	%%include/linux/kernel_stat.h%%