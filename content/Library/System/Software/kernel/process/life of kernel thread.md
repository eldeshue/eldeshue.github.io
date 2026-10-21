---
Date: 2026-10-11
tags:
  - Linux
  - Process
---
# Overview
커널스레드의 생성과 소멸에 대해서 정리한다.
# Contents
## what?
커널 내부의 함수를 **독립적으로 실행**하기 위해 존재하는 스레드. 백그라운드에서 실행되며 매니지먼트성 작업을 수행함.

- `kthreadd`: 다른 커널 스레드의 생성을 관리함, **pid=2**
- `kworker/*`: 커널의 비동기 작업용 스레드, 흔히 생각하는 worker thread
- `ksofirqd/*`: 지연 처리를 수행하는 softirq용 스레드
- `kswapd`: 메모리 관리를 위해 page를 회수, swap 수행, cache page 버림, etc
- kcompactd: 메모리 단편화를 줄이는 스레드, 일종의 조각모음

커널 스레드도 pid, tid, `task_struct`를 가지며, **user task와 동등하게 스케줄링됨**

> 유저스페이스로 돌아가지 않고, 항상 커널 모드(arm의 EL1)을 유지하는 task

> exception이 아닌, **스케줄러에 의해 실행되는 커널 코드**

> 커널 스레드는 별도의 분리된 address space를 소유하지 않음
## why?
일반적으로 커널의 코드는 exception의 형태로 실행됨. system call이나 IRQ 등이 바로 그것임. 하지만 이러한 exception만으로 일반적인 시스템을 운영할 수 없음.

> 만약 메모리 할당을 수행할 때에만 disk swap을 수행한다고 가정하면, 할당 시점에 disk IO가 필요해지고, 지연이 매우 커질 수 있음. 
> -> kswapd 같은 별도 스레드를 돌려서 미리 free page를 모아두면 예방 가능

이렇듯 별도 thread의 형태로 작업을 수행해두는 것이 큰 이득이 됨.

> 즉, IRQ나 syscall 등의 요청은 짧게 대응하고, 오래 걸리는 작업은 kernel thread로 오프로딩
> -> 비용이 발생하는 시점을 조정
> -> **요청 시점의 반응성 증가**
## state - sleep, stop, park
커널 스레드는 보통 3개의 상태를 가짐.
![595](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(237%2C%20237%2C%20237\)%22%20viewBox%3D%220%200%20330%20190%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Cg%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%3Crect%20x%3D%22116%22%20y%3D%229%22%20width%3D%2298%22%20height%3D%2230%22%20rx%3D%227%22%20fill%3D%22%2331856f%22%20fill-opacity%3D%22.18%22%20stroke%3D%22%2331856f%22%2F%3E%3Ctext%20x%3D%22165%22%20y%3D%2228%22%20font-weight%3D%22600%22%3ERUNNING%3C%2Ftext%3E%3Crect%20x%3D%2212%22%20y%3D%2293%22%20width%3D%22102%22%20height%3D%2234%22%20rx%3D%227%22%20fill%3D%22%236e82ac%22%20fill-opacity%3D%22.15%22%20stroke%3D%22%236e82ac%22%2F%3E%3Ctext%20x%3D%2263%22%20y%3D%22114%22%3ESLEEPING%3C%2Ftext%3E%3Crect%20x%3D%22216%22%20y%3D%2293%22%20width%3D%22102%22%20height%3D%2234%22%20rx%3D%227%22%20fill%3D%22%236e82ac%22%20fill-opacity%3D%22.15%22%20stroke%3D%22%236e82ac%22%2F%3E%3Ctext%20x%3D%22267%22%20y%3D%22114%22%3EPARKED%3C%2Ftext%3E%3Crect%20x%3D%22116%22%20y%3D%22153%22%20width%3D%2298%22%20height%3D%2230%22%20rx%3D%227%22%20fill%3D%22%23cb5555%22%20fill-opacity%3D%22.13%22%20stroke%3D%22%23cb5555%22%2F%3E%3Ctext%20x%3D%22165%22%20y%3D%22172%22%3EEXITED%3C%2Ftext%3E%3C%2Fg%3E%3Cg%20fill%3D%22none%22%20stroke%3D%22currentColor%22%20stroke-width%3D%221.3%22%20opacity%3D%22.65%22%3E%3Cpath%20d%3D%22M135%2039L75%2093%22%2F%3E%3Cpath%20d%3D%22M55%2093L139%2039%22%2F%3E%3Cpath%20d%3D%22M195%2039L255%2093%22%2F%3E%3Cpath%20d%3D%22M274%2093L191%2039%22%2F%3E%3Cpath%20d%3D%22M165%2039L165%20153%22%20stroke-dasharray%3D%224%204%22%2F%3E%3C%2Fg%3E%3Cg%20fill%3D%22currentColor%22%20font-size%3D%229%22%3E%3Ctext%20x%3D%2245%22%20y%3D%2254%22%3Esleep%3C%2Ftext%3E%3Ctext%20x%3D%2281%22%20y%3D%2272%22%3Ewake%3C%2Ftext%3E%3Ctext%20x%3D%22256%22%20y%3D%2254%22%3Epark%3C%2Ftext%3E%3Ctext%20x%3D%22217%22%20y%3D%2272%22%3Eunpark%3C%2Ftext%3E%3Ctext%20x%3D%22173%22%20y%3D%22142%22%3Estop%3C%2Ftext%3E%3C%2Fg%3E%3C%2Fsvg%3E)
- sleep: 작업이 없음. 현재 상태를 보존하고, cpu를 반납.
- park: 외부의 요청으로 일시 정지.
- stop: 스레드의 종료 절차 수행. 다시 부활할 수 없음.

## Birth - kthread_create
`kthreadd` 스레드가 생성을 총괄함. `kthread_create*` api 호출을 통해서 `kthreadd`에 **생성을 요청함.**

``` c
// workqueue의 worker thread 예시
worker->task = kthread_create_on_node(worker_thread, worker, pool->node, "kworker/%s", id_buf);
```
- worker_thread: 스레드 핸들러, 실제 동작
- kworker: 커널 스레드의 이름, ps에서 검색되는 그것
``` c
// irq thread 예시
// bottom-half의 threaded irq 처리를 위한 스레드
// 내부적으로 kthread_create_on_node를 호출하는 매크로
t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
```
- irq_thread: 스레드 핸들러
- new: 스레드 핸들러에 전달할 인자
### kernel thread 생성 절차
1. kthread_create 류의 생성 요청을 받음
2. kthreadd를 스케줄링, wkake-up
3. kthreadd가 **kernel_clone -> copy_process** 호출

즉, 커널 스레드 생성도 최종적으로는 동일한 api인 `copy_process`로 생성.
## Death
스레드마다 생애 주기가 다름.

최종적으로는 `do_exit()`을 통해서 공통적으로 종료한다. 이는 커널 입장에서 유저 스레드나 커널 스레드나 동등한 `task_struct`로 관리하기 때문.

``` c
// kthread.h, line 119
#define kthread_exit(result) do_exit(result)
```
다만 독립된 주소공간인 `mm_struct`가 없어서 일부 다른 점이 있음.
### 1. return 0
커널 함수가 return하며 종료하는 경우. `kthreadd`가 반환 값을 받아서 do_exit 처리함. 
### 2. kthread_stop(task)
다른 스레드에서 종료를 요청하는 방식. **직접 제거하는 것이 아님**.
요청을 받기로 되어있는 스레드 쪽에서 다음과 같이 직접 상태를 확인하고 종료해야 함.
``` c

static int my_thread(void *data)
{
    while (!kthread_should_stop()) {
        // 작업 수행
        msleep(1000);
    }

    return 0;
}
```
외부 요청을 받아서 종료한다는 점에서는 SIGKILL과 유사한 위치인듯.
# Summary

# Reference
