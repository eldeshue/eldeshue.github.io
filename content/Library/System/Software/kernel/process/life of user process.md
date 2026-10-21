---
Date: 2026-10-08
tags:
  - Linux
  - Process
---
# Overview
유저 프로세스의 생성과 소멸에 대해서 다룬다. 
# Contents
## Birth - kernel_clone
커널에서 프로세스 생성의 실제 구현은`kernel_clone`이다. 과거에는 `_do_fork` 였다.

> 흔히 사용하는 `fork()`는 user space에서 접근하기 위한 wrapper임

대략적인 시스템 콜 호출 흐름은 다음과 같다.
```
-> kernel_clone:실제 동작
-> _arm64_sys_clone:시스템 콜 핸들러 
-> invoke_syscall
-> context switching, handler table 참조
fork:유저 호출점
```

> 시스템 콜 핸들러는 arch depend하다.

> 시스템 콜 핸들러의 시그니쳐는 [[macro in kernel|SYSCALL_DEFINE 매크로]]로 정의, 테이블에 등록된다.

### `_arm64_sys_clone`

### `kernel_clone`
``` c
pid_t kernel_clone(struct kernle_clone_args *args)
{
...
	p = copy_process(NULL, trace, NUMA_NO_NODE, args);

	wake_up_new_task(p); // 생성 직후 프로세스 실행, 런큐에 큐잉

}
```
- wake_up_new_task : 생성 직후 한 번 즉시 실행.

### `copy_process`
부모프로세스의 리소스를 복사. 흔히 알려진 fork의 실체라 할 수 있음.
``` c
...
	p = dup_task_struct(current, node); // task_struct 생성, 실행할 stack 포함
	if (!p)
		goto fork_out;
		
...
	retval=copy_creds(p, clone_flags); // credential 정보 복사
	
	retval=sched_fork(clone_flags, p); // 스케줄링 관련 속성 복사
	
	retval=secrity_task_alloc(p, clone_flags); // 보안 정보 복사, lsm 등
	
	...
```

- proc가 실행할 stack이 필요, 이를 할당해줌
- credential은 권한 정보
- clone_flags를 바탕으로 복사 수행 여부를 판단.

복사되는 속성은 대략 다음과 같음.
- fd
- siganl
- vaddr 관련  자원(vma).
- etc, 굉장히 많음.

> 신규프로세스는 fork를 외친 부모의 정보를 상속받음.
## Death - exit
프로세스는 다음의 두 가지 경우에서 종료한다.
- exit() 호출
- SIGKILL, 9
### exit()

```
호출 경로: 
 
-> do_exit // 핵심 종료 루틴
-> do_group_exit 
-> __wake_up_parent // __arm64_sys_exit_group?
-> invoke_syscall
-> context switching, handler table 참조
user 영역 exit 호출
```

 ftrace로 본 callstack에서 `__wake_up_parent`가 `__arm64_sys_exit_group`대신 등장하는데, 이는 `do_group_exit`의 return address를 잘못 해석한 결과임.
 
 사실 `do_group_exit`은 프로세스 종료로 인해서 return하지 않음. 하지만 arm의 함수 호출 명령어인 `bl`은 return address를 갖고, 디버거가 bl의 return address에 해당하는 자리에 우연히던 값을 읽어서 심볼로 변환한 결과가 `__wake_up_process` 였을 뿐.
### SIGKILL
시그널에 의한 프로세스 종료. 값은 9.

프로세스는 인터럽트 등으로 인해서 kernel mode에 들어온 후, **user mode로 복귀하기 직전**에 시그널을 **수신**한다.

> 시그널은 3단계의 과정을 거쳐서 처리된다.
> 1. signal_generate: 발생, 특정 task를 대상으로 하는 시그널을 생성
> 2. signal pending: 시그널의 전달을 위해서 커널이 기록
> 3. signal_deliver: 해당 시그널을 목적 task가 인지하고, 그에 따른 핸들러 준비 혹은 기본 동작

> kill 시그널의 기본 동작은 종료이므로, **kernel mode에서 수행**된다. 

> 사용자 등록 시그널 핸들러는 **유저모드**에서 실행된다.

signal delivery가 커널모드에서 복귀하는 직전에 실행하는 이유는 signal handler를 user mode에서 처리하기 위함이다. 따라서, 커널은 복귀 시점에 signal 유무를 확인하고 signal handler 실행을 위한 context 보존 및 stack을 준비하고 모드를 전환한다. 

```
-> do_exit
-> do_group_exit
-> do_signal
-> get_signal
-> do_notify_resume
-> el0_svc
```
다만, SIGKILL은 기본 동작이 process 종료이며, 호출되는 구현은 `do_group_exit`, `do_exit`로 동일하다.

# Summary

# Reference