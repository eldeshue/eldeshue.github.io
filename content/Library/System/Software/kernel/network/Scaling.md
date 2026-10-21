---
Date: 2026-08-24
tags:
  - Linux
  - Network
---
# Overview
패킷 처리량 증진을 위해서 리눅스 네트워크 스택이 제공하는 코어 관련 최적화 프레임워크.

사용자로 하여금 네트워크 스택 하단의 패킷 처리 방식을 조정하는 엔지니어링 포인트를 제공한다.
# Contents
## RSS: Receive Side Scaling 
**nic가 읽어온 패킷을 받아낼 큐를 선택**하는 매커니즘. 

nic는 큐를 단위로 패킷 처리를 수행한다. 큐를 단위로 DMA를 수행하며, 따라서 RSS는 nic의 패킷 처리에 대한 병렬화다. 동시에 nic의 hard irq는 보통 큐 단위로 이루어지므로, nic의 RX를 처리할 코어에 대한 병렬화이다. 즉, nic와 그 nic에서 읽어올 코어로 패킷 처리를 병렬화한다.

> RSS는 네트워크 스택 최하단의 수신 경로를 병렬화한다. 

큐의 결정을 위해서는 tuple을 hashing한 값을 사용하는데, 이를 flow hash라 한다.
## RPS: Receive Packet Steering
**flow를 바탕으로 패킷을 처리할 코어를 선택**하는 프레임워크. flow 식별을 위해서 마찬가지로 flow hash를 사용한다.

RPS의 핵심은 패킷을 처리할 코어를 하드웨어 레이어(IRQ 등)과 분리한다는 점에 있다. 앞서 RSS에 의해 napi가 스케줄링되고, hard IRQ에 의해 최초 처리 코어가 결정된다. 즉, RPS가 없다면, RSS에서 설정한 nic 처리 코어들에서만 패킷이 처리된다. 따라서, RPS는 flow hash에 맞는 코어로 한 번 더 load balancing을 수행하여 실제 네트워크 스택 처리를 한 번 더 병렬화 한다.

> RSS가 nic 수준의 패킷 처리를 병렬화 한다면, RPS는 커널 수준에서 패킷 처리를 병렬화 한다.

부수적으로 동일한 flow를 최대한 같은 코어가 처리하도록 유도하여 locality를 증진시키는 효과가 있다. 

> RPS는 Flow에 기반한 core의 load balancing이며 동시에 locality 증가 효과도 기대할 수 있다.

다만, 한 코어 내에서 특정 flow의 패킷이 몰릴 경우, 그 외의 flow가 처리되지 못하는 불균형이 발생할 수 있다. 이를 해결하기 위해서 flow limit이라는 매커니즘이 있어, 정도를 넘는 flow의 패킷은 의도적으로 drop하여 다른 flow를 보호한다.
## RFS: Receive Flow Steering
**패킷 flow의 소켓을 처리하는 스레드의 코어로 패킷을 처리**하도록 하는 매커니즘이다.

해당 패킷을 실제로 소비한 유저스레드와 동일한 코어로 패킷을 처리하는 것으로, RPS와 마찬가지로 locality를 증진시키기 위함인데, 보다 유저 스페이스의 친화적인 매커니즘이라고 생각한다.

패킷의 유저스페이스 소비는 보통 소켓을 통해 이루어지므로, 소켓 IO가 처리된 스레드로 스티어링된다고 할 수 있다.

> RFS는 스택과 유저 스레드를 같은 코어에서 실행, 유저 레벨의 패킷 처리를 최적화한다.

> 개인적으로는 얼마나 유용한지 의심스럽다.
# Summary
- 3가지 기술은 각자 분리된 영역에서 최적화를 제공한다.
	- RSS는 하드웨어, nic
	- RPS는 커널
	- RFS는 user thread, socket
# Reference
- https://docs.kernel.org/networking/scaling.html