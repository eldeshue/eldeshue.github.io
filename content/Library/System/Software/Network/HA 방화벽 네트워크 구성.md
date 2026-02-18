---
tags:
  - Network
  - Firewall
Date: 2026-02-10
---
# Overview
이중화된 방화벽을 중심으로 한 간소화된 네트워크 구성을 정리한다.
# Content
## What is HA?
HA는 High Availability의 약자로, 고가용성이라 번역된다.

고가용성이란 많은 처리량에도 불구하고, 시스템이 무너지지 않도록 하기 위해 fail over에 대한 대처가 되어 있음을 의미한다.

이러한 HA는 많은 곳에서 다양한 형태로 나타나는데, 네트워크의 경우 packet flow를 처리하기 위해서 다중화의 형태를 띈다.

## 방화벽

### Virtual IP


이를 바탕으로 스위치 설정도 다음과 같이 필요하다.
- **Promiscuous Mode (혼잡 모드):** 자신에게 할당되지 않은 MAC 주소를 가진 패킷도 수신할 수 있어야 한다.
- **MAC Address Changes:** VM의 유니캐스트 MAC 주소 변경을 허용해야 한다.
- **Forged Transmits (위조된 전송):** VIP를 사용하는 과정에서 발생하는 가짜 소스 MAC 주소 패킷의 전송을 허용해야 한다.

## Network 구성
방화벽 네트워크이므로 망 분리가 필요하다.
### 모델 - Gateway
해당 네트워크 구성에서 방화벽은 gateway의 역할을 한다.

클라이언트는 서버의 주소를 알고 보내지만, 해당 경로 상에 방화벽이 존재하며, 그 방화벽이 패킷 필터링을 실시한다.
### Internal
trusted, 방화벽으로 보호, 서버
### External
untrusted, 방화벽 바깥의 영역, 클라이언트
### HA
방화벽들로 구성된 네트워크
## 