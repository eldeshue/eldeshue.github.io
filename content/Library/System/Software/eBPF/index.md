---
title: eBPF
Date: 2026-02-08
tags:
  - Linux
  - eBPF
---
# Definition
eBPF는 linux kernel에서 제공하는 kernel space 런타임이다.

> eBPF는 extended Berkeley Packet Filer의 약자다. 현재는 기존의 이름과는 상관이 없다.
# How it works?
eBPF의 핵심 아이디어는 사용자가 개발한 **임의의 바이너리를 커널이 제공하는 진입점에 삽입, 커널 스페이스에서 실행하는 것**이다. 여기서 커널은 마치 **VM처럼** 동작한다.
## 1. eBPF byte code
먼저 개발자는 ebpf로 실행할 코드를 준비한다. 코드는 보통 C나 Rust로 구성되며, 어떤 system call에 어떤 방식으로 load될 것인지 명시하게 된다. 

이 코드는 **llvm을 통해서 eBPF를 target으로 하는 바이트 코드로 컴파일**된다. 

> 마치 GPU 프로그래밍의 shader와 유사하다.
## 2. eBPF Loader
앞서 준비한 바이트코드를 커널에 삽입하기 위한 별도의 loader 프로세스가 필요하다. loading과정은 **bpf system call**로 수행되며, 삽입 위치 및 이를 위한 권한(capability)을 확인하게 된다.

삽입 위치를 결정하기 위해서 다음의 세 방법이 존재한다.
1. tracepoint: 커널 빌드 시점에 활성화되는 eBPF용 진입점. static hook.
	- 가장 효율적.
	- 하지만 빌드 시점에서 미리 설정이 필요.
2. probe: interrupt를 활용한 실행. 특정 인스트럭션을 복제해둔 다음 interrupt로 변경한다. 실행 중 해당 interrupt를 실행하면, 그 핸들러로 지정해둔 ebpf 바이너리를 실행한다. 일명 dynamic hook.
	- 가장 자유롭게 삽입 위치를 결정할 수 있다.
	- interrupt를 사용하기에 가장 느리다.
3. trampoline: 특정 함수의 시작과 끝에 no-op을 삽입한 다음, ebpf 삽입 시 해당 no-op을 jump로 변경, eBPF를 실행한다. 즉, 런타임에 활용 가능한 일반화된 static hook이라 할 수 있다.
	- 거의 모든 시점에 자유로이 삽입 가능.
	- 성능상의 단점 거의 없음. 심지어 인자 전달이 더욱 효율적임.
	- BTF 필수.
> 더 자세한 내용은 [[BTF]]에서 별도로 다룬다.
## 3. Jit Compile
커널은 SW이고, 그 실행 환경은 매우 다양(intel, amd, arm, etc)하다. 따라서 eBPF 바이트 코드는 bpf 시스템 콜에 의한 loading 과정에서 **실제 target machine에 맞는 바이너리로 컴파일** 되어야 한다. 즉, **Just In Time 컴파일**이 발생한다.

여기서 또 중요한 것은, 커널에서 실행되어야 하기에 eBPF 코드는 절대 실패해선 안된다는 것이다. 따라서 eBPF는 엄격한 제약(stack size, 무한 루프 여부, etc)이 존재한다. 그리고 이러한 제약을 검사하는 linux의 기능이 바로 **verifier**이다. 모든 eBPF 바이트코드는 verifier를 통과해야 loading될 수 있다.
## 4. Execution
이렇게 삽입된 eBPF 바이너리는 시스템이 수행하며 거치는 여러 hook에 대해 호출되며 동작을 수행하게 된다. 이 eBPF는 bpf 시스템 콜 호출 시점에서 받은 fd가 살아있는 동안 지속된다.

eBPF의 가장 일반적이고 유명한 활용 방법은 monitoring 툴이다. kernel space의 여러 지점에 유저 코드를 삽입하므로, 실재 커널의 동작을 모니터링하기에 아주 좋다.

이밖에도 kernel을 거치는 도중에 packet을 읽어낼 수 있어서 고성능 Network IO가 가능하다.
# Why eBPF?
eBPF는 다음과 같은 여러 이유로 System Programming의 핵심 기법으로 떠올랐다.
## Safety
기존의 커널 스페이스 실행을 위한 방법은 크게 두 가지가 있었다. 바로 커스텀 커널(커널 코드를 직접 수정 후 빌드)과 커널 모듈(커널에 런타임 삽입 가능)이 바로 그것이다. 하지만 둘 모두 오류가 발생하면 바로 panic으로 이어져 시스템의 안정성을 극히 위협했다.

ebpf는 이와 달리, 오류가 발생하는 경우 verifier에 의해 걸러지며, 실행 도중 실패한다 해도 해당 **프로세스가 죽을 뿐 커널이 멈추진 않는다**.
## Programmability
eBPF의 또 다른 핵심 장점은 programmability, 즉 구현 가능성이다. 기존의 LKM(Loadable Kernel Module)이나 커스텀 커널의 경우에는 제약이 존재했다. 

LKM의 경우 정해진 형식이 존재해 그 형식을 벗어날 수 없었으며, 커스텀 커널의 경우 linux가 요구하는 구현 명세가 있어 그 구현 방향에 한계가 존재했다. 심지어 커스텀 커널의 경우 빌드를 새로이 해야 하므로, 상당한 시간이 걸리기까지 한다.

하지만 eBPF의 경우, 단순 삽입과 해제가 자유롭고, 삽입 위치도 다양하며, eBPF Maps를 사용하여 user space와 소통까지 가능하다. 이런 점에서 eBPF는 압도적인 개발의 자유로움을 갖는다.
## Efficiency
커널 스페이스 런타임이므로, 유저 스페이스로의 컨텍스트 스위칭이 발생하지 않기에 높은 효율을 갖는다. 기존의 구현의 경우에는 핵심 데이터만 유저 스페이스로 전송하고, 그 데이터 처리는 유저 스페이스에서 수행해야 했으며, 이는 context switching이 필수적임을 의미했다.

> 이러한 효율성은 네트워크 패킷 모니터링 등에서 압도적인 효율을 보여준다.

---
# Reference
- 커뮤니티 공식 소개 문서 : https://ebpf.io/ko-kr/what-is-ebpf/