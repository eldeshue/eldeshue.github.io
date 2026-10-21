---
Date:
tags:
  - Linux
  - Network
---
# Overview
리눅스 네트워크 스택에서 패킷의 receive, 즉 RX flow를 정리한다.

여기서는 nic가 읽어온 패킷이 `napi`에 의해 schedule되고, `GRO`를 거쳐서 본격적으로 네트워크 스택에 진입하는 시점인 `__netif_receive_skb_core`를 다룬다.
# Contents
## __netif_receive_skb_core


# Summary

# Reference