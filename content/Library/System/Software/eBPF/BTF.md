---
Date: 2026-02-08
tags:
  - Linux
  - eBPF
---
# Overview
앞서 eBPF의 [[./index|간단한 이론적 배경]]을 알아보았다. 여기서는 eBPF의 고등 활용의 핵심이 되는 BTF에 대해 다룬다.
# Content
## What is it?
BTF는 BPF Type Format의 약자로, **커널을 구성하는 모든 타입(구조체)의 layout 정보**를 표현한 것으로, `/sys/kernel/btf`아래에 위치한다.
## Why? - CO-RE
커널에는 보안상의 이유로 booting시점에 메모리 레이아웃을 random으로 memory layout을 뒤섞는 기능이 들어있다.  

### vmlinux.h
