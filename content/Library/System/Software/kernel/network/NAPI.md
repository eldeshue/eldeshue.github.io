---
Date: 2026-08-02
tags:
  - Linux
  - Network
---
# Overview
NAPI는 리눅스가 네트워크 디바이스의 IO를 위해 제공하는 공통 프레임워크다.
# Contents
## Mental Model
### IO: Bottom Half
네트워크 디바이스도 어느 디바이스와 마찬가지로 IRQ가 발생하면 IO를 처리한다. 그러나 순수 IRQ만으로 IO를 처리하기에는 다음과 같은 문제가 있다. 

- Interrupt Storm: 매 packet에 대해 IRQ가 발생하면 context switching이 너무 빈번하게 발생하며, 이 switching 과정에 너무 많은 비용이 발생한다.
- Starvation: 패킷 처리에 몰두하느라 user space가 코어를 받지 못한다.
- IRQ context가 너무 길어짐: IRQ 핸들링 맥락은 여러가지 제약이 존재하며, 이 핸들링 과정이 길어지면 코어 분배가 공정하지 않게 되어버린다.

이런 문제를 해결하기 위해서 bottom-half 매커니즘을 도입한다. 

> bottom-half는 IRQ 핸들링을 hard/soft 두 맥락으로 나누어 수행하는 것이다.
> hard: 짧고 중요한 작업만 실제 인터럽트 핸들러 맥락(hard-IRQ)에서 수행한다.
> soft: 실제 처리가 필요한 복잡한 작업은 일반 커널 실행 맥락에서 수행한다.

패킷 처리에 대입하면 다음과 같다.

1. 일정한 수준 이상의 패킷이 nic에 의해 읽히면, nic는 IRQ를 발동한다.
2. hard-IRQ: 발동한 현재 해당 인터럽트를 봉인(mask))하고, 패킷 처리를 예약하고 종료한다.
3. soft-IRQ: 핸들러 종료 즉시 패킷 처리를 수행하며, 정해진 budget을 바탕으로 다음을 수행한다.
	- budget을 모두 소모: 계속해서 패킷이 전달되고 있다. 해당 nic를 polling하며 패킷 처리를 계속한다. `napi_struct`의 `weight` 필드가 바로 그것.
	- budget이 남음: 한동안 패킷이 전달되지 않음. 인터럽트를 다시 활성화(unmask)하고, 종료.

즉, NAPI는 네트워크 디바이스의 **IO 방식(얼마나, 어떻게)을 추상화**한다.
### Core Mapping
NIC는 내부적으로 큐를 가지며, 이 큐에는 메모리 버퍼가 있어, 송수신을 위한 패킷 데이터를 저장한다. 수신(RX)과 송신(TX)은 별도의 큐로 이루어지며, 구성 방식에 따라서 여러 쌍을 가질 수 있다.

> 이 큐를 다른 말로 ring이라고 부른다.

패킷을 처리한다는 것은 이 큐를 소비하는 것이며, 이를 위해서 모든 드라이버는 `poll`함수를 정의한다. 

> poll함수가 soft-IRQ에서 호출되어 RX 큐의 패킷을 처리, TX 큐의 전송 완료된 메모리를 회수한다.

네트워크 디바이스가 리눅스에서 초기화 될 때, IO 수를 결정하기 위해서 MSI-X 인터럽트 벡터, 인터럽트 핸들러가 등록되고, 실제 큐를 소비할 `poll`함수는 `napi_struct`를 통해 시스템에 등록된다.

즉, 드라이버 개발자는 NAPI로 `poll`함수를 등록하는 것을 통해서, 하나의 코어(soft-IRQ)가 몇 개의 큐를 소비할 것인지 결정한다.

또한 드라이버 개발자는 IRQ 핸들러에서 몇 번의 `napi_schedule`을 호출하느냐에 따라서 IRQ가 몇 개의 코어를 스케줄링할 지 결정한다.

> 일반적으로 현대 nic는 **RX/TX큐 페어, IRQ, 코어, napi 인스턴스**의 비율을 **1:1:1:1**로 가져간다. 

정리하자면 다음과 같다.
- nic는 패킷 처리를 위한 큐를 갖는다.
- 드라이버는 이 큐를 소비하기 위해 `poll`메서드를 제공한다.
- 시스템은 `napi_struct`단위로 bottom-half를 수행하여 `poll`의 실행을 결정한다.
## Implementation
### napi_struct
``` c
/*
 * Structure for NAPI scheduling similar to tasklet but with weighting
 */
struct napi_struct {
	/* The poll_list must only be managed by the entity which
	 * changes the state of the NAPI_STATE_SCHED bit.  This means
	 * whoever atomically sets that bit can add this napi_struct
	 * to the per-CPU poll_list, and whoever clears that bit
	 * can remove from the list right before clearing the bit.
	 */
	struct list_head	poll_list;

	unsigned long		state;
	int			weight;      // RX를 처러 완료 목표 패킷 수
	u32			defer_hard_irqs_count;
	int			(*poll)(struct napi_struct *, int); // 큐 소비 함수
#ifdef CONFIG_NETPOLL
	/* CPU actively polling if netpoll is configured */
	// 특정 스레드를 직접 할당하여 napi polling을 수행
	int			poll_owner;
#endif
	/* CPU on which NAPI has been scheduled for processing */
	int			list_owner;
	struct net_device	*dev; // 호출한 디바이스 정보
	struct sk_buff		*skb;
	struct gro_node		gro;
	struct hrtimer		timer;
	/* all fields past this point are write-protected by netdev_lock */
	struct task_struct	*thread;
	unsigned long		gro_flush_timeout;
	unsigned long		irq_suspend_timeout;
	u32			defer_hard_irqs;
	/* control-path-only fields follow */
	u32			napi_id;
	struct list_head	dev_list;
	struct hlist_node	napi_hash_node;
	int			irq;
	struct irq_affinity_notify notify;
	int			napi_rmap_idx;
	int			index;
	struct napi_config	*config;
};

```

- poll함수가 `napi_struct`를 인자로 받는다, 즉 `napi_struct` 자체가 큐 소비의 context이다.
- per cpu한 `poll_list`가 있어, `napi_struct`자신을 전달한다.
- bottom half 매커니즘으로 작업을 미뤄서 수행하는 측면에서 `taskelet`과 그 구현이 유사하다. 하지만 napi의 경우 `weight`이라는 예산이 존재한다는 점이 다르다.
# Summary
- 네트워크 디바이스의 bottom-half IO의 주체가 바로 napi.
- nic 드라이버 개발자에게 IO에 대한 엔지니어링 포인트를 제공한다.
# Reference  