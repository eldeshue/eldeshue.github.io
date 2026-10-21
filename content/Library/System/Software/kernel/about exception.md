---
Date: 2026-10-11
tags:
  - Linux
---
# Overview
운영체제 이론에서 등장하는 인터럽트 관련 여러 중요 개념 사이의 관계에 대해서 정리하고 넘어간다.

- exception
- IRQ
- soft IRQ
- system call
- software interrupt
# Contents
## exception
예외, 순차적으로 실행되기를 기대하는 실행 흐름에서 벗어나는 모든 종류의 행위를 이름.
따라서 이후에 다루게 될 모든 용어를 포함하는 가장 큰 개념임.

> 기존 실행 흐름에서 벗어나는 모든 것.
## IRQ
인터럽트. cpu외의 **하드웨어가 발생시키는 신호**에 의한 exception.
하드웨어가 신호를 보내면, 모드 전환이 수행되고, 커널이 irq 핸들러 실행을 위해서 동작함.

기존의 실행 맥락과 아무런 관련이 없는, exception이므로, **비동기(async)적**이다.

> cpu 외의 하드웨어가 보내는 신호에 의한 예외.

> 타이머, net dev, char dev, etc
## softirq
softirq는 linux 고유의 매커니즘으로, 인터럽트 처리를 나중에 실행하도록 예약하는 방식.

softirq는 커널모드 복귀 경로 혹은 `ksoftirqd`등을 통해서 처리됨.

> software interrupt랑 아무런 관련이 없음.

> 이전 NAPI에서 본 그것
## System call
코드 실행 중, 커널에 의한 처리를 요청하는 것. arm64의 `svc`, x86에서는 `syscall` 등의 특수한 instruction에 의해서 유발됨.

실행 과정에서 발생하는 exception이라 **동기(sync)적**이다.

> fork, exit, etc
## software interrupt - 구형 x86의 전통적 표현
 과거 x86에서는 INT 명령어로 system call을 수행하였음. 그 구현이 **Interrupt Description Table, IDT**를 사용하는 방식이었는데, 그로 인해서 software interrupt라는 표현을 사용했다.

즉, **진입 방법의 유사성**으로 인해서 관련 표현이 존재했던 것이다.
 
하지만 현재 x86-64 리눅스에서는 주로 `syscall`을 사용하므로, 해당 표현은 적절하지 않다.

## Syscall vs IRQ
개인적으로 이 두 요소의 구현 방향에 대해서 상당한 오해를 가지고 있었고, 여기서 그 차이를 정리한다.

핵심 차이는 **task_struct의 유무**이다. Syscall은 task이고, IRQ는 task가 아니다(일부 예외가 있으니 후술). 그리고 task_struct는 스케줄러에 대한 주권이다. task를 갖는 주체만이 스케줄레의 선택을 받는다. 

Syscall은 호출한 user task의 맥락에서 실행된다. 따라서 Syscall handler의 실행은 비록 모드는 커널(EL1)이어도 user task의 일부이고, 따라서 스케줄링이 가능하다. 그리고 스케줄링이 가능하기에 기본적으로 `sleep()`, `schedule()`이 가능하다.

한 편 IRQ는 다르다. IRQ는 하드웨어에 의해서 유발되며, 일시적으로 cpu를 빼앗아서 실행되는 존재다. IRQ는 기존 cpu가 실행하던 task와 아무런 관련이 없다. task는 맥락을 보존하여 별도로 저장되고, IRQ는 별도의 context, 별도의 stack에서 실행된다.

IRQ는 task가 없으므로, `sleep()`, `schedule()`을 호출하여 cpu를 포기하면 실행 맥락을 복원할 방법이 없다. 따라서 일반적인 IRQ에서는 관련 호출이 금지된다.

> IRQ는 스케줄링을 통해서 실행되지 않으므로 별도의 task를 갖지 않는다. 

> 그러나 예외적으로 kernel thread 위에서 실행되는 threaded IRQ가 존재한다.

다만, 앞서 software interrupt에서 다룬 것처럼 과거에는 진입 방식에 유사성이 있었다.
# Summary
- 가장 큰 개념은 exception, 기존 실행 흐름을 이탈함.
- IRQ: 발생 원인이 하드웨어의 신호, 비동기적
- System Call: 특정 명령어 실행, 동기적

# Reference