---
Date: 2026-10-08
tags:
  - Linux
---
# Overview

커널 디버깅을 위한 여러 도구에 대해서 정리한다.

- Crash-Utility
- TRACE32

# Content

## Crash Utility

GNU 계열의 kernel 디버거.

주로 시스템의 특정 시점(kernel panic 등)의 kernel 상태를 해석하는 디버거.

- vmlinux: elf, dwarf 정보 제공, **반드시 버젼 일치해야 함**
- vmcore: 메모리 덤프, ram의 snapshot, 생성 시점의 ram 정보()
    - kdump로 수집, makedumpfile로 필터링 & 압축
    - 필터링 설정에 의해 free page, page cache 등 일부 정보 제거, 용량 감소

> 하드웨어의 상태에 대해서는 잘 모를 수 있음. 주로 가상 메모리 기반임.

### ps

당시의 프로세스의 목록을 출력한다.

> pid, ppid, cpu, **task_struct의 주소**, 상태(task status), mem, name(comm)

- -l : 실행 시간순 정렬, timestamp 제공, last arrival
- -p : process의 상속 트리 출력, 어디서 기원했는지 추적해볼 수 있음
- -G : 사용 메모리 정보, leak 등을 확인 가능

> 시스템의 ps가 procfs에 기반한다면, crash의 ps는 vmcore에서 task를 읽어서 재구성함

## TRACE32

실행 중인 시스템을 **멈추고**, 실제 **CPU와 물리 RAM** 에서 얻은 정보로 프로세스 해석하는 디버거.

CPU의 디버깅 전용 기능을 활용하여 시스템을 멈춤, vmlinux의 커널 정보를 활용, 리눅스 시스템 전체의 상태를 해석한다.

> ram에서 시스템 정보 역산: 물리 주소에서 시작, freeze 당시의 가상 주소(mmu, page table)로 변환 실제 돌던 cpu를 멈춘거라 상태가 오염 가능(외부는 실행하고 있으므로)

### syscall 호출

```
__do_sys_epoll_pwait(inline) <- 호출된 syscall의 구현
__se_sys_epoll_pwait(inline)

__arm64_sys_epoll_pwait(:regs=0x...) <- 실제 syscall handler, arch depend

__invoke_syscall(inline)
invoke_syscall(:regs=0x...,:scno=22,:sc_nr-451,:
el0_svc_common(regs=0x...,scno=22, sc_nr=?,syscal
do_el0_svc(:regs=0x...)
el0_svc(...)
el0t_64_sync_handler(regs=?)
el0t_64_sync(asm)
# 전형적인 syscall 호출 흐름

```

> 콜스택은 아래에서 위로 syscall handler는 arch dependent, __arm prefix 존재

일반적인 디버깅과 달리 syscall 내부 흐름이 보이는 이유는 **실제 실행중인 CPU & RAM**의 정보를 읽어왔기 때문. 실행중인 stack을 읽어서 kernel symbol을 매핑해서 읽음

### 커널 스레드(kworker/u9:0)

```
__switch_to(...)
context_switch(...)
__schedule(...)
schedule()
worker_thread(...) <- 스레드 핸들러, 커널 스레드 동작의 구현부
kthreadd(...)
ret_from_fork(asm)
NSX:0x...(asm)

```

> kworker는 work queue 관련 커널 스레드 user space의 콜 구조(el0, invoke_syscall, etc)과는 확연히 다르다

# Summary

## 용도

- 일반 gdb: 살아있음, MRI, CT, 청진기, ...
- crash: 죽었음, 해부
- TRACE32: 죽였다가 살림, 급속냉동/해동

## 대상

- 일반 gdb: 보통 단일 바이너리(user proc)
- crash: kernel
- TRACE32: system 전체

## 정보 구성

모두 elf / dwarf에서 static 정보 획득

동적인 정보는

- 일반 gdb: ptrace(), kernel이 알려줌
- vmcore: 당시의 kernel ram snapshot, 일부 생략됨
- TRACE32: 실행중인 cpu / ram