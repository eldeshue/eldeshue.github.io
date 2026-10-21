---
Date: 2026-10-20
tags:
  - Linux
  - Process
---
# Overview
리눅스 커널이 다루는 프로세스의 실체에 해당하는 구조체, `taks_struct`에 대해 다룬다.
# Contents

``` c
// include/linux/sched.h
// line 819
struct task_struct {
	...
}
// line 1668 ....
```

>  온갖 kconfig로 인해 구조체 길이가 무려 800줄..... 모두 아는 것은 불가능.....
## 1. thread_info
**task_struct의 첫 번째 필드**, 프로세스의 arch 의존적인 상태 정보를 표현한다. 
``` c
// X86
struct thread_info {
	unsigned long flags; /* low level flags */
	u32 status; /* thread sync flags */
}

// arm32, v7
struct thread_info {
	unsigned long flags; /* low level flags */
	int preempt_count; /* 0 이면 선점 가능, 0보다 크면 불가, 0보다 작으면 버그 */
	mm_segment_t addr_limit // stack 주소 한계
	struct task_struct *task; // owner의 주소
	__u32 cpu; // 
}

// arm64
// kconfig에 따라서 필드 달라질 수 있음.
struct thread_info {
	unsigned long flags; /* low level flags */
	// ttbr0 pan 옵션을 위한 ttbr0 값 추가 가능
	u64 preempt_count; // 32비트로 쪼개서 절반은 count, 나머지 절반은 상태값 표현
	// shadow call stack 옵션을 위한 base, sp 포인터 추가 가능
	u32 cpu;
}
```


> 특정 구현 및 캐싱 효율을 위해서 반드시 task_struct의 첫 필드로 존재해야 함.

첫 필드가 된 이유는 다음과 같음

![585](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(237%2C%20237%2C%20237\)%22%20viewBox%3D%220%200%20330%20218%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%2236%22%20y%3D%2212%22%20width%3D%22185%22%20height%3D%22187%22%20rx%3D%225%22%20fill%3D%22none%22%20stroke%3D%22currentColor%22%20stroke-width%3D%221.5%22%2F%3E%3Crect%20x%3D%2237%22%20y%3D%2213%22%20width%3D%22183%22%20height%3D%22138%22%20fill%3D%22%236399c3%22%20fill-opacity%3D%22.14%22%2F%3E%3Crect%20x%3D%2237%22%20y%3D%22152%22%20width%3D%22183%22%20height%3D%2246%22%20fill%3D%22%23dc9844%22%20fill-opacity%3D%22.22%22%2F%3E%3Cpath%20d%3D%22M36%20152%20H221%22%20stroke%3D%22currentColor%22%20stroke-opacity%3D%22.45%22%2F%3E%3Ctext%20x%3D%22128%22%20y%3D%2271%22%20text-anchor%3D%22middle%22%20font-size%3D%2213%22%20fill%3D%22currentColor%22%3EKernel%20Stack%3C%2Ftext%3E%3Ctext%20x%3D%22128%22%20y%3D%2289%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%20opacity%3D%22.65%22%3E%EC%8A%A4%ED%83%9D%EC%9D%80%20%EC%95%84%EB%9E%98%EB%A1%9C%20%EC%84%B1%EC%9E%A5%20%E2%86%93%3C%2Ftext%3E%3Ctext%20x%3D%22128%22%20y%3D%22177%22%20text-anchor%3D%22middle%22%20font-size%3D%2213%22%20fill%3D%22currentColor%22%3Ethread_info%3C%2Ftext%3E%3Cpath%20d%3D%22M225%20174%20H250%22%20stroke%3D%22currentColor%22%20stroke-width%3D%221.2%22%2F%3E%3Ctext%20x%3D%22256%22%20y%3D%22173%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3Etask%3C%2Ftext%3E%3Ctext%20x%3D%22256%22%20y%3D%22185%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3E%ED%8F%AC%EC%9D%B8%ED%84%B0%3C%2Ftext%3E%3Ctext%20x%3D%2236%22%20y%3D%22212%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%20opacity%3D%22.7%22%3E%EC%A0%95%EB%A0%AC%EB%90%9C%20%EC%8A%A4%ED%83%9D%EC%9D%98%20%EC%8B%9C%EC%9E%91%20%EC%A3%BC%EC%86%8C%3C%2Ftext%3E%3C%2Fsvg%3E)

- 과거 thread_info는 stack에 있었음. 이는 stack overflow로 손상되는 보안 문제 있었음
	- `CONFIG_THREAD_INFO_IN_TASK`로 task_struct로 이동
- `task_struct`의 타입 정보를 몰라도 `thread_info`를 사용하기 위함 ( [메일](https://openwall.com/lists/kernel-hardening/2016/09/15/14?utm_source=chatgpt.com))
	- 커널 헤더 순환 의존성 문제, `taks_struct`정의 없이 `thread_info`가 필요한 구현 존재
	- `thread_info` 는 arch dependent 구현
	- low 레벨의 구현에서 `thread_info`를 쓰기 위해 `task_struct`를 알아야 하는 순환 문제 발생
``` c
// include/linux/thread_info.h

#ifdef CONFIG_THREAD_INFO_IN_TASK
/*
 * For CONFIG_THREAD_INFO_IN_TASK kernels we need <asm/current.h> for the
 * definition of current, but for !CONFIG_THREAD_INFO_IN_TASK kernels,
 * including <asm/current.h> can cause a circular dependency on some platforms.
 */
#include <asm/current.h>
#define current_thread_info() ((struct thread_info *)current)
#endif
```

``` c
// arch/arm64/include/asm/preempt.h
static inline void clear_preempt_need_resched(void)
{
	current_thread_info()->preempt.need_resched = 1;
}
```

---

### `preempt_count` - 프로세스 컨텍스트 정보
컨텍스트의 핵심이 되는 필드인 `preempt_count`. 현재 실행 중인 context를 표현한다.
``` c
u64 preempt_count = 0x0000000000000

0x100 // Soft IRQ
0x10000 // IRQ context
0x1 // preemption disable
```
특정 shift 범위로 context를 구분하고, 각 범위에 대한 산술 연산을 통해서 context별 depth를 추적한다. context에 진입하면 증가, 이탈하면 감소한다. 직접 수정하지 않고, 준비된 api를 통해서 조작한다.

필드 구조는 다음과 같다.
``` c
union {
	u64 preempt_count;
	struct {
		u32 ;
		u32 ;
	};
}
```
arm64를 기준으로, 해당 값은 union을 통해서 32비트씩 나눠서 절반은 count, 나머지 절반은 resched 등의 상태 요도로 사용한다.

전체 필드를 종합하여 0이면 scheduling 가능으로 판단한다. 따라서 매우 중요한 값.

> 디버깅의 핵심 정보
## 2. `__state`
**두 번째 필드**, 프로세스 상태 머신의 그 상태 필드.

- `TASK_RUNNING` : 실행중 or 스케줄링 가능, **0**
- `TASK_INTERRUPTIBLE` : sleep에 진입, IRQ 가능, **1**
- `TASK_UNINTERRUPTIBLE` : 특정 조건을 대기하는 sleep에 진입, IRQ 불가능(특정 조건 대기), **2**

> `__` prefix가 붙었음은 컨벤션 상 직접적인 값 변경을 '지양'하고 'api' 사용을 권장

> 해당 경우에는 `set_current_state()` 호출

``` c
// drivers/crypto/mxs-dcp.c
// static int dcp_chan_thread_aes()

set_current_state(TASK_UNINTERRUPTIBLE);

if(!backlog && !arq) {
	schedule(); // cpu를 반납하고 sleep
	continue;
}

set_current_state(TASK_RUNNING);

```
- schedule()을 호출하여 cpu를 포기하기 전후로, `__state`를 변경해주고 있음.
- 일종의 lock_guard? RAII 패턴인듯

이하는 state 전환 api 구현

``` c
#define set_current_state(state_value) \
	do { \
		debug_normal_state_change((state_value)); \
		smp_store_mb(current->__state, (state_value)); \
	} while(0)
```
`smp_store_mb` 메모리 베리어를 자동으로 삽입하고 있음. 이는 메모리 barrier 삽입하여 메모리 오더링 문제를 방지하기 위함.

> 덤프 기반 디버깅의 핵심 정보, 커널로그->런큐->**프로세스 state**

## 3. `saved_state`
``` c
	/* saved state for "spinlock sleepers" */
	unsigned int			saved_state;
```
RT 리눅스용으로 존재하는 필드, Freezer라는 기능을 위해서 `PREEMPT_RT`옵션이 없는 일반 커널에서도 활성화됨.

RT 환경에서는 spin lock을 잡아도 sleep해버릴 수 있음. 이 필드는 spin lock을 잡고 sleep하는 경우를 위한 상태 보존 필드임.

> RT는 Real Time의 약자로, 리눅스를 RTOS와 유사하게 동작하도록 하는 옵션
## 4. `randomized_struct_fields_start`
``` c
	/*
	 * This begins the randomizable portion of task_struct. Only
	 * scheduling-critical items should be added above here.
	 */
	randomized_struct_fields_start
```
보안을 위해서 구조체 필드를 랜덤화 하는 `RANDSTRUCT`의 적용 범위를 설정하는 경계. KASLR과 유사한 무언가로 보임.

> KASLR: 커널 코드 및 데이터의 메모리 배치 랜덤화, 런타임(부팅)에 결정

> RANDSTRUCT: 구조체의 레이아웃 랜덤화, 컴파일 타임에 결정

왜 앞서 살펴본 3개 필드는 랜덤화 되지 않을까? 앞선 3개 옵션은 스케줄러가 빈번하게 건드리는, scheduling-critical한 값들이다. 따라서 이 필드들은 **`task_struct`의 주소에서 가장 가까운 위치에 고정 배치하여 캐싱 효율성을 높이기 위함**.

`thread_info`의 경우 앞서 설명한 이유로 랜덤화에서 제외됨.
# Summary

# Reference
- `thread_info`의 첫 필드 강제 제약 관련 커밋: https://openwall.com/lists/kernel-hardening/2016/09/15/14?utm_source=chatgpt.com
