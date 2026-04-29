---
Date: 2026-04-27
tags:
  - Linux
---
# Overview
리눅스 커널 빌드 과정에 대해서 다룬다.
# Contents
## 소스 획득
다음의 경로에서 커널 소스를 얻을 수 있다. 
``` bash
# 6.18 버전을 예로 하면...
# v6.12 버전을 예시로 합니다. (원하는 버전으로 수정 가능)
git clone --depth 1 --branch v6.18 https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```
depth 옵션을 1로 하여 불필요한 히스토리를 제거, 용량을 줄인다.
![[kernel_root_dir.png]]
## kconfig
리눅스 커널은 무수히 많은 설정을 가지며, 여러 필수 설정(타겟 하드웨어 아키텍쳐) 및 편의성 설정(디버깅을 위한 심볼, 메모리 카나리, 성능 측정용 옵션 등) 등을 포함한다.

이들 설정은 매크로의 형태로 구현되며, 이를 autoconf.h로 관리한다.
``` c
#define CONFIG_FEATURE_NAME1 1
#define CONFIG_FEATURE_NAME2 1
#define CONFIG_FEATURE_NAME_MODULE3 1
```
이 헤더파일은 빌드 과정에서 자동으로 생성되는데, 이를 위해서는 .config파일이 필요하다.

커널 설정을 구성하는 .config파일은 다음과 같은 형태로 설정을 표현한다.
``` 
CONFIG_OPTION_NAME1=y
CONFIG_OPTION_NAME2=n
CONFIG_OPTION_NAME3=m
...
```
- n은 설정하지 않아서 완전히 배제함을 의미한다. 
- y는 해당 설정을 vmlinux에 정적으로 링킹됨을 의미한다.
- m은 별도의 파일로 빌드됨을 의미한다. 별도의 .ko 파일로 빌드되는 타겟을 생성하며, 추후 빌드로 동적으로 로드될 수 있다. **vmlinux의 크기를 줄인다.**
	- m으로 설정한 옵션들은 별도로 빌드하여 나중에 추가해줘야 함
![[config_example.png]]

직접 파일을 작성할 수 있지만, gui로 편하게 설정할 수도 있다. 이를 위해 repo의 루트 Makefile은 .config를 생성하는 gui를 위한 타겟을 제공한다. 
- gconfig, menuconfig, xconfig
![[menuconfg_gui_image.png]]
_e.g) menuconfig_

대부분의 경우 타겟 하드웨어 아키텍쳐에 맞게 미리 설정된 config 파일을 변형해서 사용한다. 이를 위한 타겟이 defconfig 이며, 다음과 같이 아키텍쳐마다 존재한다. 

![[arch_dir.png]]
![[arm_defconfig_example.png]]
아키텍쳐별 preset인 defconfig도 존재한다.
## 빌드
config파일을 구성했다면, 빌드를 수행할 수 있다.
### 패키징 - 아키텍쳐 의존성 해결
모든 커널 빌드의 결과는 vmlinux이다. 그러나, 리눅스 커널은 현존하는 거의 모든 하드웨어에서 돌아가기 위해서 아키텍쳐에 의존성을 필요로 한다.

> **vmlinux** : 커널 빌드의 최종 결과물, 압축되지 않은 ELF 파일이다. 모든 아키텍쳐에서 공통으로 생성된다. 용량이 매우 크기에 부팅을 위한 가공이 필요하다.

이를 해결하기 위해서 커널의 최상위 make는 빌드 시점에 아키텍쳐 관련 코드의 make를 포함시킨다.
``` Makefile
include $(srctree)/arch/$(SRCARCH)/Makefile
```

이렇게 결정된 Makefile은 vmlinux를 생성한 다음, 아키텍쳐별 형태로 vmlinux를 가공(패키징)한다. 다음은 패키징된 리눅스 커널의 타겟 이름이다.

``` bash
# 커널 빌드 명령어
make zImage -j$(nproc)       # ARM 표준
make bzImage -j$(nproc)      # x86 표준, big zImage
...
```

크로스 컴파일의 경우, 툴체인을 추가로 명시해줘야 한다.
![[arch_arm_make_zimage.png]]
_arch/arm/Makefile_
![[arch_arm_boot_make_zimage.png]]
_arch/arm/boot/Makefile_
### vmlinux
커널 빌드 이미지인 vmlinux는 여러 컴포넌트로 구성된다. 이들은 kconfig에 의해 선택되며, 각자 독자적인 디렉토리를 가지고 고유의 Makefile로 빌드되고, built-in.o라는 오브젝트에 링크되어 vmlinux를 구성한다. 

> **최신 리눅스 커널에서는 증분 빌드를 위해 build-in.o 대신 built-in.a를 사용한다.** 
### zImage
부팅을 위해 vmlinux를 압축하고, 부트스트랩 코드를 추가한 것이다. 이미지를 압축한 이유는 압축 해제가 디스크IO보다 빠르기 때문이다.

- 부트스트랩 코드 : head.o, misc.o
- 압축된 vmlinux : piggy.o

> 부트스트랩 코드: 커널 이미지를 압축 해제하기 위한 코드, 부트로더가 호출한다.
### System.map
vmlinux와 함께 만들어지는 텍스트 파일로, 커널 내의 모든 심볼의 메모리 맵 역할을 한다. 

> **덤프의 주소값을 번역하는 용도로 사용된다.**

### 커널 헤더
커널에 추가하기 위한 외부 모듈(드라이버 등)을 위해서는 커널 헤더 파일이 필요하다. 
headers_install 타겟으로 헤더를 얻을 수 있다.
# Summary
- 특정 원하는 기능이 있는 버젼의 커널 소스를 구한다.
- 커널 빌드를 위해서는 .config로 설정을 명시한다. 설정에는 다양한 방법이 존재한다.
- 커널 빌드의 결과는 vmlinux이다.
- 커널 부팅을 위해서는 vmlinux를 가공할 필요가 있으며, zImage등 다양한 결과물이 존재한다.
- 부팅은 하드웨어 밀접한 동작이기에 arch 아래의 하드웨어 디펜던트한 타겟에 의존한다.
- 빌드 과정은 다음과 같다.
	1. 준비: .config 해석 및 autoconf.h 생성
	2. vmlinux 빌드: 각 디렉토리 빌드 및 링킹, vmlinux 획득
	3. 패키징: arch 디렉토리로 이동, 압축 및 부트스트랩 코드와 합쳐서 패키징 수행
# Reference
- 코드로 알아보는 ARM 리눅스 커널