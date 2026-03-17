발생 빈도가 높거나 빠르게 처리해야할 때 사용
	핸들러 호출 직후 실행
	동적으로 실행하도록 제공되는 [[tasklet]] 인터페이스가 존재

네트워크 패킷처리나 고속 그래픽 처리, 스토리지(UFS) 드라이버를 구현
드라이버에서 요청한 동적커널 타이머들은 타이머 인터럽트 이후 softirq로 실행된다

softirq 서비스 요청을 받으면 이를 처리하는 과정

%%kernel/softirq.c%%
```
const char * const softirq_to_name[NR_SOFTIRQS] = {
	"HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "IRQ_POLL",
	"TASKLET", "SCHED", "HRTIMER", "RCU"
};
```

위의 서비스들은 부팅시 open_softirq() 함수를 통해 등록된다

요청시 raise_softirq() 함수를 호출
do_softirq() 함수에서 서비스 실행
	soft irq 서비스 핸들러를 호출하여 실행한다

실행흐름 [[interrupt 시 컨텍스트 흐름]]
1. 인터럽트 핸들러에서 soft irq 서비스를 요청
2. 인터럽트서비스 루틴 종료 시 irq_exit() 함수에서 softirq 유무 점검
3. 서비스 요청 확인 시 `__do_softirq()` 호출
4. 종료 시 다시 softirq 서비스 유무 다시 검토
	1. softirq 처리시간이 2ms 이상 or 10번 이상 수행했다면
			wakeup_softirq호출하여 ksoftirqd 프로세스를 깨움
			`__do_softirq 종료
5. ksoftirqd 프로세스가 마무리 못한 soft irq 서비스 실행


