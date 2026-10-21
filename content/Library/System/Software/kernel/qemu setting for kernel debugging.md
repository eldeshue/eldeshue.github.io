---
tags:
  - Linux
Date: 2026-10-08
---
# Overview
리눅스 커널 디버깅 실습을 위한 qemu 세팅에 대해서 다룬다.
# Contents
## 0. packages
``` bash
sudo dnf install \
    qemu-system-aarch64 \
    gcc-aarch64-linux-gnu \
    binutils-aarch64-linux-gnu \
    git \
    make \
    bc \
    bison \
    flex \
    openssl-devel \
    elfutils-libelf-devel \
    ncurses-devel
```
## 1. qemu
arm64 커널 실습을 위한 기본 세팅은 다음과 같다.

``` bash

qemu-system-aarch64 \
    -machine virt \
    -cpu cortex-a72 \
    -accel tcg \
    -smp 4 \
    -m 2G \
    -kernel path/to/kernel/image \
    -append "console=ttyAMA0 root=/dev/vda rw" \
    -drive file=path/to/rootfs.ext4,format=raw,if=virtio \
    -nographic \
    -s \
    -S

```
최대한 Raspberry Pi와 유사한 환경을 목표로 했다. 

- -machine virt: arm64용 가상 보드
- -cpu cortex-a72: raspberry-pi와 동일한 cpu
- -nographic: serial 통신
- -append: 커널의 콘솔 접속용 tty 정보
- -drive: rootfs, mount용
- -smp: 코어의 수, 편리한 디버깅을 위해서는 1로 enfrjt
- -s & -S: gdb를 위한 stub 옵션 & 실행 즉시 정지

> 현재 실행 환경이 x86_64 머신인 관계로, kvm가속은 쓰지 못하고 cpu emul 수행

## 2. 커널 빌드
먼저 기본 config 생성을 위한 기본 설정을 진행.
``` bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export BUILD=/path/to/build/tree

make O=$BUILD ARCH=arm64 defconfig
```

이후 debugging용 필수 옵션을 다음과 같이 활성화 한다.
```
./scripts/config --file $BUILD/.config \
    -e DEBUG_KERNEL \
    -e DEBUG_INFO \
    -e DEBUG_INFO_DWARF4 \
    -d DEBUG_INFO_DWARF5 \
    -e GDB_SCRIPTS \
    -e FRAME_POINTER \
    -e KALLSYMS \
    -e KALLSYMS_ALL \
    -d RANDOMIZE_BASE \
    -e KEXEC \
    -e KEXEC_FILE \
    -e CRASH_DUMP \
    -e MAGIC_SYSRQ \
    -e DEBUG_FS \
    -e IKCONFIG \
    -e IKCONFIG_PROC \
    -e FTRACE \
    -e FUNCTION_TRACER \
    -e FUNCTION_GRAPH_TRACER
```
이하는 디버깅 강화용 옵션
```
./scripts/config --file $BUILD/.config \
    -e DEBUG_LIST \
    -e DEBUG_MUTEXES \
    -e DEBUG_SPINLOCK \
    -e DEBUG_ATOMIC_SLEEP \
    -e PROVE_LOCKING
	
```
- kernel 디버깅 옵션(최적화 방지 및 dwarf symbol 포함)
- gdb 디버깅용 헬퍼
- ftrace 활성화
- KASLR **비활성화** ->커널 addr 랜덤화로 디버깅 곤란해질 수 있음
- 그 외 여러 보안 기능 비활성화

concig에 옵션을 확실히 반영하기 위해서 다음 실행.
```
make O=$BUILD ARCH=arm64 olddefconfig
```

이후 최종 빌드를 다음과 같이 실행
```
make \
    O=$BUILD \
    ARCH=arm64 \
    CROSS_COMPILE=aarch64-linux-gnu- \
    -j$(nproc)
```
이후 빌드 트리로 지정한 dir에서 image와 vmlinux 바이너리가 생성되며, image는 qemu에게, vmlinux는 gdb등에게 제공하면 된다.

이후 추가적으로 gdb 활용을 위한 helper도 빌드한다.
```
make \
    O=$BUILD \
    ARCH=arm64 \
    CROSS_COMPILE=aarch64-linux-gnu- \
    scripts_gdb
```
## 3. busybox 빌드
qemu로 커널을 부팅할 계획이므로, 이제는 커널의 rootfs의 binary가 되어줄 busybox를 빌드한다.

> busybox는 단일 바이너리로 여러 cli를 동시에 수행하는 특수한 바이너리이다. 

최대한 간단한 rootfs를 구성하기 위해서 의존성을 최소화 할 계획이다. 동적 링킹을 최대한 배제하기 위해서 musl libc를 사용, busybox를 arm64, single static binary로 빌드한다.

musl은 static linking을 고려한, 의존성 최소화 툴체인으로, 지금과 같은 간단한 arm64 환경의 user space binary 빌드에 적합하다.

> 단, arm64커널은 aarch64-linux-gnu 툴체인으로 빌드한다.
## 4. Initramfs 구성
user space는 rooftfs가 필요하다. 일반적으로 이 rootfs는 디스크에 마운트되지만, 부팅 초기에는 disk driver를 비롯한 많은 것이 부족하여 disk가 아닌 ram을 block device 삼아서 mount를 하는데, 이를 initramfs라 한다. 

pid1 스레드는 initramfs를 ram에 unzip하고 마운트, init을 실행하는데, 보통 이 init에서 최종 rootfs로 전환한다. 그러나, 우리의 목표는 커널의 실행에 있으므로, initramfs로 충분하다.
## 5. qemu 실행
앞서 소개한 커맨드를 실행, 인스턴스를 실행한다.
# Summary

# Reference