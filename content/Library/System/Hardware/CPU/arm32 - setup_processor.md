---
Date:
tags:
  - Linux
  - ARM
---
# Overview
리눅스 부팅 과정 중, arm32비트 프로세서의 초기화에 대해서 다룬다.

> start_kernel -> setup_arch -> setup_processor

# Contents
## 1. cpu 정보 획득하기

``` c
// 6.18
static void __init setup_processor(void)
{
	unsigned int midr = read_cpuid_id();
	struct proc_info_list *list = lookup_processor(midr);
```
먼저 read_cpuid_id()를 통해서 cpu id를 얻어온다. 32비트 정수로, arm cpu 고유 값이다.

``` c
// arch/arm/include/asm/cputype.h
#ifdef CONFIG_CPU_CP15
#define read_cpuid(reg)							\
	({								\
		unsigned int __val;					\
		asm("mrc	p15, 0, %0, c0, c0, " __stringify(reg)	\
		    : "=r" (__val)					\
		    :							\
		    : "cc");						\
		__val;							\
	})
    ...

```
arm 프로세서 중, 코프로세서가 존재하는 arm 프로세서의 경우, cp15 특수 레지스터에서 cpuid를 읽어온다. 
``` c
#elif defined(CONFIG_CPU_V7M)
...
#define read_cpuid(reg)							\
	({								\
		WARN_ON_ONCE(1);					\ // 방어용 코드, cortex m은 cp15가 없음.
		0;							\
	})
```
arm v7 cortex M 프로세서의 경위, 이런 코프로세서가 없는 mcu용도의 프로세서로, 해당 코드를 실행할 경우, 경고 메시지를 내도록 되어있다.

---

``` c
/*
 * locate processor in the list of supported processor types.  The linker
 * builds this table for us from the entries in arch/arm/mm/proc-*.S
 */
struct proc_info_list *lookup_processor(u32 midr)
{
	struct proc_info_list *list = lookup_processor_type(midr);

	if (!list) {
		// 탐색 실패, 무한 루프로 에러처리!!!
		pr_err("CPU%u: configuration botched (ID %08x), CPU halted\n",
		       smp_processor_id(), midr);
		while (1)
		/* can't use cpu_relax() here as it may require MMU setup */;
	}

	return list;
}
```
이후 획득한 cpuid값을 바탕으로, 커널 바이너리 내에 존재하는 프로세서 정보 테이블을 순회, 해당 cpuid에 맞는 프로세서의 정보를 저장한 proc_info_list를 가져온다.

---
#### 왜 무한 루프로 에러를 처리하나?

- 정상 종료? : 현재 보드가 무엇인지 모르므로, 전원 끄는 방법을 모름. 
    - 전원 관리를 위해서는 별도의 펌웨어나 전원 관리 칩에 대한 접근이 필요함
- panic? : 현재 cpu 정보 모름(초기화 실패), console 모름, reboot 모름
- 부트로더로 복귀? : 이미 메모리를 한 번 갈아 엎었음, 부트로더가 기대하는 상태가 아님
- 계속 진행한다? : MMU 등 필수 기능 할 줄 모름

> 아무것도 할 수 없음, 무한 루프가 안전한 대처

> 에러처리도 기능이다!!!
## 2. 전역 cpu 정보 초기화

``` c
    // setup_processor()
	cpu_name = list->cpu_name;
	__cpu_architecture = __get_cpu_architecture();

	init_proc_vtable(list->proc);
```
앞서 조회한 `proc_info_list` 를 바탕으로 전역 cpu 정보 초기화를 수행한다.

---

``` c
// 단순 아키텍쳐 분기
#ifdef CONFIG_CPU_V7M
static int __get_cpu_architecture(void)
{
	return CPU_ARCH_ARMv7M;
}
#else
// 범용 프로세서
static int __get_cpu_architecture(void)
{
	int cpu_arch;

	if ((read_cpuid_id() & 0x0008f000) == 0) {
		cpu_arch = CPU_ARCH_UNKNOWN;
	} else if ((read_cpuid_id() & 0x0008f000) == 0x00007000) {
		cpu_arch = (read_cpuid_id() & (1 << 23)) ? CPU_ARCH_ARMv4T : CPU_ARCH_ARMv3;
	} else if ((read_cpuid_id() & 0x00080000) == 0x00000000) {
		cpu_arch = (read_cpuid_id() >> 16) & 7;
		if (cpu_arch)
			cpu_arch += CPU_ARCH_ARMv3;
	} else if ((read_cpuid_id() & 0x000f0000) == 0x000f0000) {
		/* Revised CPUID format. Read the Memory Model Feature
		 * Register 0 and check for VMSAv7 or PMSAv7 */
		unsigned int mmfr0 = read_cpuid_ext(CPUID_EXT_MMFR0);
		if ((mmfr0 & 0x0000000f) >= 0x00000003 ||
		    (mmfr0 & 0x000000f0) >= 0x00000030)
			cpu_arch = CPU_ARCH_ARMv7;
		else if ((mmfr0 & 0x0000000f) == 0x00000002 ||
			 (mmfr0 & 0x000000f0) == 0x00000020)
			cpu_arch = CPU_ARCH_ARMv6;
		else
			cpu_arch = CPU_ARCH_UNKNOWN;
	} else
		cpu_arch = CPU_ARCH_UNKNOWN;

	return cpu_arch;
}
#endif
```
cpuid 값에 대해서 마스킹된 비트를 조회하여 기능의 활성화 여부를 확인한다.

cortex M의 경우, 아무런 처리를 하지 않는다.

---
``` c
#if defined(CONFIG_BIG_LITTLE) && defined(CONFIG_HARDEN_BRANCH_PREDICTOR)
#include <linux/smp.h>
extern struct processor *cpu_vtable[];
#define PROC_VTABLE(f)			cpu_vtable[smp_processor_id()]->f
#define PROC_TABLE(f)			cpu_vtable[0]->f
static inline void init_proc_vtable(const struct processor *p)
{
	unsigned int cpu = smp_processor_id();// 호출 시점의 cpu의 id를 획득
	*cpu_vtable[cpu] = *p;
	WARN_ON_ONCE(cpu_vtable[cpu]->dcache_clean_area !=
		     cpu_vtable[0]->dcache_clean_area);
	WARN_ON_ONCE(cpu_vtable[cpu]->set_pte_ext !=
		     cpu_vtable[0]->set_pte_ext);
}
#else
```
  proc_vtable을 호출하는 이유는 다음과 같다.
  
 1. 선점
	- sm_processor_id를 부른 다음 스케줄링이 수행되면, 동기화 문제가 발생
	- 따라서 per cpu한 vtable을 미리 초기화 해두는 것으로 문제를 원천 해결
	 
  2. big/little 2중 코어
	- 프로세서들이 서로 다른 구현(big, little)을 가질 수 있음
	- 각자 다른 값으로 테이블 초기화 가능

특히 위 함수의 경우, 부팅 과정에서만 호출되는 것이 아닌, smp 구조에서 여러 프로세서에 대해서 호출된다. 따라서 각 프로세서별 초기화가 필요하다.
## 3. cpu 정보 출력
``` c
// setup_processor()
	pr_info("CPU: %s [%08x] revision %d (ARMv%s), cr=%08lx\n",
		list->cpu_name, midr, midr & 15,
		proc_arch[cpu_architecture()], get_cr());   

```
앞서 초기화 한 cpu의 정보를 출력한다.
## 4. 플랫폼 정보 초기화

``` c
// setup_processor()
// user의 elf를 위해 제공할 cpu 기능 정보가 elf_hwcap
	snprintf(init_utsname()->machine, __NEW_UTS_LEN + 1, "%s%c",
		 list->arch_name, ENDIANNESS);  // uname->machine, 프로세서 아키텍쳐 출력 
	snprintf(elf_platform, ELF_PLATFORM_SIZE, "%s%c",
		 list->elf_name, ENDIANNESS);   // elf 위한 플랫폼(프로세서, v7 등)정보 초기화
	elf_hwcap = list->elf_hwcap; // 기능 표현 비트 초기화, simd 가능 여부, 정수 나눗셈 가속 등

	cpuid_init_hwcaps();    // cpu id를 읽고, 그에 맞는 elf를 위한 capacity 초기화
	patch_aeabi_idiv();     // arm 프로세서 중 일부는 정수 나눗셈 integer divide를 지원 안함, 관련 구현

    // 커널 config가 관련 기능 지원 안함, 앞서 활성화한 기능 지움
#ifndef CONFIG_ARM_THUMB
	elf_hwcap &= ~(HWCAP_THUMB | HWCAP_IDIVT);
#endif

    // 캐시 속성의 기본 정책 초기화
#ifdef CONFIG_MMU
	init_default_cache_policy(list->__cpu_mm_mmu_flags);
#endif

    // 특정 cpu 설계 결함 회피
	erratum_a15_798181_init();

	elf_hwcap_fixup();  // elf 관련 hwcap 보정
```
플랫폼 정보라고는 하지만, 사실 프로세서 정보이다.

주로 elf 바이너리 포맷에 노출시키기 위한 프로세서 정보를 초기화 한다. 

elf 포맷은 여러 특수한 목적(glibc, JVM, etc)를 위해서 현재 프로세서의 정보를 ABI의 형태로 바이너리에게 노출시켜줄 필요가 있다. 이를 위한 내용을 `elf_hwcap` 등의 형태로 표현하는데, 여기서는 이들을 초기화하고 있다.
## 5. 캐시 초기화

``` c

static void __init cacheid_init(void)
{
	unsigned int arch = cpu_architecture(); // 앞서 호출한 __get_cpu_architecture() 결과 반환

    // 아키텍쳐에 따른 분기
	if (arch >= CPU_ARCH_ARMv6) {
		unsigned int cachetype = read_cpuid_cachetype();    // read_cpuid(CPUID_CACHETYPE), 레지스터에서 읽어오기
        // cachetype에 따른 분기
		if ((arch == CPU_ARCH_ARMv7M) && !(cachetype & 0xf000f)) {
			cacheid = 0;
		} else if ((cachetype & (7 << 29)) == 4 << 29) {
			/* ARMv7 register format */
			arch = CPU_ARCH_ARMv7;
			cacheid = CACHEID_VIPT_NONALIASING;
			switch (cachetype & (3 << 14)) {
			case (1 << 14):
				cacheid |= CACHEID_ASID_TAGGED;
				break;
			case (3 << 14):
				cacheid |= CACHEID_PIPT;
				break;
			}
		} else {
			arch = CPU_ARCH_ARMv6;
			if (cachetype & (1 << 23))
				cacheid = CACHEID_VIPT_ALIASING;
			else
				cacheid = CACHEID_VIPT_NONALIASING;
		}
		if (cpu_has_aliasing_icache(arch))
			cacheid |= CACHEID_VIPT_I_ALIASING;
	} else {
		cacheid = CACHEID_VIVT;
	}

```
프로세서의 기능을 바탕으로 cache 기능을 확성화 한다. arm 프로세서는 그 구현에 따라서 다양한 캐시 정책을 갖는데, 이는 필히 커널에게 알려져야 마땅하다. 위 코드에서는 이를 식별하고 있다.

> 보다 상세한 내용은 [[arm cache 정책 구조]]를 참고.
## 6. cpu 초기화

``` c
// 각 프로세서의 자기 자신을 위한 전용 실행 환경 초기화 공통 함수
/*
 * cpu_init - initialise one CPU.
 *
 * cpu_init sets up the per-CPU stacks.
 */
void notrace cpu_init(void)
{
#ifndef CONFIG_CPU_V7M
	unsigned int cpu = smp_processor_id();  // 현재 프로세서 정보
	struct stack *stk = &stacks[cpu];   // per cpu stack 인스턴스

	if (cpu >= NR_CPUS) {
		pr_crit("CPU%u: bad primary CPU number\n", cpu);
		BUG();
	}

     // per cpu 스택 등 전용 영역을 빠르게 찾기 위한 offset 초기화
     // 현재는 부팅중 호출되었으므로, 특수한 동작을 함
     // 이후 smp_prepare_boot_cpu 호출로 multi processor가 가능해지면 그 때 다시 호출, 정상 동작
	set_my_cpu_offset(per_cpu_offset(cpu));

    // 벤더가 제공하는 실제 cpu 초기화 수행
    // #define cpu_proc_init			PROC_VTABLE(_proc_init) // 앞서 init_proc_vtable에서 초기화한 init 호출
	cpu_proc_init();    
    ...
#endif

```
앞서 초기화 한 per cpu 테이블의 값을 바탕으로, 프로세서 초기화를 수행한다.

---
### 커스텀 스택 초기화
``` c
    // exception 발생 시 사용할 per cpu 전용 스택 셋업
	/*
	 * setup stacks for re-entrant exception handlers
	 */
	__asm__ (
	"msr	cpsr_c, %1\n\t"
	"add	r14, %0, %2\n\t"
	"mov	sp, r14\n\t"
	"msr	cpsr_c, %3\n\t"
	"add	r14, %0, %4\n\t"
	"mov	sp, r14\n\t"
	"msr	cpsr_c, %5\n\t"
	"add	r14, %0, %6\n\t"
	"mov	sp, r14\n\t"
	"msr	cpsr_c, %7\n\t"
	"add	r14, %0, %8\n\t"
	"mov	sp, r14\n\t"
	"msr	cpsr_c, %9"
	    :
	    : "r" (stk),
	      PLC_r (PSR_F_BIT | PSR_I_BIT | IRQ_MODE),
	      "I" (offsetof(struct stack, irq[0])),
	      PLC_r (PSR_F_BIT | PSR_I_BIT | ABT_MODE),
	      "I" (offsetof(struct stack, abt[0])),
	      PLC_r (PSR_F_BIT | PSR_I_BIT | UND_MODE),
	      "I" (offsetof(struct stack, und[0])),
	      PLC_r (PSR_F_BIT | PSR_I_BIT | FIQ_MODE),
	      "I" (offsetof(struct stack, fiq[0])),
	      PLC_l (PSR_F_BIT | PSR_I_BIT | SVC_MODE)
	    : "r14");
#endif
}
```
프로세서 초기화를 수행했으면, 프로세서의 특수 모드에 따른 전용 스택을 만든다.

이 스택은 `stack`구조체로 정의되고, 각 cpu의 특정 버젼마다 존재한다.

모드는 다음과 같다.
- IRQ : Interrupt ReQuest, 하드웨어 인터럽트
- ABT : ABorT, divide by 0를 비롯한 진짜 에러의 경우
- UND : UNDefined instruction, 미정의 동작
- FIQ : Fast Interrupt reQuest, 특수한 인터럽트
- SVC : SuperVisor Call, 소프트웨어 인터럽트, 일반적인 커널 코드는 해당 모드로 실행된다. 따라서 별도의 스택을 초기화 하지 않고, 해당 모드로 전환만 수행한다.

이들 특수한 모드를 위한 전용 스택이 필요한 이유는 일반적인 스택을 공유할 경우, 데이터에 오염이 발생할 우려가 있기 때문이다. 

이런 전용 스택에서는 주로 현재 프로세서의 레지스터 프로파일 복사 및 서비스 루틴으로의 분기 등을 수행한다.
# Summary

1. CPU ID register를 읽는다.
2. 커널이 지원하는 proc_info_list 항목을 찾는다.
3. 해당 CPU에 맞는 processor/TLB/cache/user operation table을 설치한다.
4. 유저 공간에 노출할 CPU capa 정보를 초기화한다.
5. cache aliasing 특성을 판별한다.
6. boot CPU의 per-CPU stack 및 CPU 초기화

> 즉, 단순 실행 중인 boot **CPU를 커널이 이해하는 형태**로 만들기
# Reference
- ARM 리눅스 커널