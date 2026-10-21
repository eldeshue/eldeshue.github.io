---
title: Linux Network Subsystem
Date:
tags:
---
# Overview
리눅스 네트워크 서브시스템을 구성하는 여러 요소에 대해 정리합니다.
# Contents
## Network Stack
### sk_buff
네트워크 통신의 단위인 패킷을 표현하는 구조체.
### NAPI
네트워크 디바이스 드라이버를 구현하기 위해서 리눅스 커널이 제공하는 범용 API.
### Scaling
커널이 제공하는 패킷 처리 최적화 프레임워크에 대해서 다룬다.
### Packet Flow - RX
네트워크 스택으로 진입하는 패킷의 경로에 대해서 알아보자.
### Packet Flow - TX
네트워크 스택에서 떠나는 패킷의 경로에 대해서 알아보자.
## Network Protocol
### VRRP
### DHCP