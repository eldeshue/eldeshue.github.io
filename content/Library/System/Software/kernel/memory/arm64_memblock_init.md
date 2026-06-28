---
Date: 2026-06-29
tags:
  - Linux
  - Memory
---
# Overview
부팅 과정에서 사용하는 원시 메모리 할당자인 memblock의 초기화를 다룬다.

> 메모리의 경우 32bit ARM나 X86-64는 레거시 구현이 많은 관계로, **ARM64 코드를 분석한다**.

> start_kernel -> setup_arch -> arm64_memblock_init

> /arch/arm64/mm/init.c

> memblock 구현 분석은 [[memblock|여기]]에서

하나 주의할 점은 해당 함수는 memblock의 region을 채우지 않는다. 모든 최초 region은 device tree를 해석하는 과정에서 이미 추가되었고, 이 함수는 kernel이 관리할 수 있는 범위로 **재단**한다.
# Contents
## 1. linear map 크기 계산
가장 먼저 linear map 영역 크기를 계산한다. 

linear map은 가상 메모리의 한 종류로, 물리 메모리와 **연속적으로 매핑**된 구간이다. 해당 가상 주소는 시작 물리 주소와 offset만 알면, 단순 덧셈으로 물리 주소를 알 수 있다.

커널은 이 linear map 영역을 특수하게 관리하며, 이를 위한 주소공간을 지정해서 운영하는데, 그 영역의 크기를 계산하는 것이다.

계산 식은 다음과 같다.
``` C
s64 linear_region_size = PAGE_END - _PAGE_OFFSET(vabits_actual);
```
단순히 linear map의 끝 주소에서 시작 주소를 빼서 그 크기를 구한다.

먼저 vabits_actual은 1워드 중, 실제 주소를 표현하는 데에 쓰는 bit의 수다. vabits_actual값이 48이라면, 64bit 중 48bit는 주소를, 나머지 16bit는 권한이나 보안 기능 구현에 사용된다.

\_PAGE_OFFSET() 매크로의 역할은, 정수를 받아서 해당 번째 비트 이상을 마스킹하는 값을 만드는것이다. 정의는 다음과 같다.
``` C
#define _PAGE_OFFSET(va) (-(UL(1) << (va)))
```
여기서 -를 곱한다는 것은, 모든 비트를 flip하고 1을 더하는 것이다. 따라서 val만큼 1을 shift한 다음, 모든 비트를 반전하고, 다시 1을 더한다.
``` C
int const val = 4;
((char)(1) << val) ==  0b00010000;
-((char)(1) << val) == 0b11101111 + 1 == 0b11110000;
```
즉, 주소를 표현하지 않는 bit 수 만큼 1로 초기화 한 값을 계산해준다.
만약 일반적인 48bit 구성이라고 하면, 시작 주소는 다음과 같다.
``` text
0b000000000000000100...0;
-> 0xffff_0000_0000_0000;
```
마찬가지 방식으로 매크로 정의된 `PAGE_END`도 계산할 수 있다.
``` C
PAGE_END == -(1 << (48 - 1)) == 0xffff_8000_0000_0000;
```
단순 계산으로는 -2^47 + 2^48이므로, 크기는 2^47이다.

이러한 방식으로 linear map 영역의 주소가 설정된 이유는 커널이 가상 주소의 특정 비트를 보고 해당 주소가 어떤 위치(유저, 커널, 커널의 특정 영역, 등)의 메모리인지 구분하기 때문이다.
## 2. 표현할 수 없는 주소의 ram 제거
CPU가 주소 범위로는 사용할 수 없는 만큼의 ram을 탑재한 경우(va bits를 초과), 초과분을 포기한다. memblock에서는 다음과 같이 제거된다.
``` C
memblock_remove(1ULL << PHYS_MASK_SHIFT, ULLONG_MAX);
```
여기서 `PHYS_MASK_SHIFT`는 유효한 물리 주소 범위(하위 비트 수)를 의미하며, 유효 범위 이상의 ULLong의 최댓값만큼(즉 모두)을 memblock에서 제외함을 뜻한다.

`memblock_remove`함수는 내부적으로 `memblock_remove_rnage`함수를 호출하며, `memblock_isolate_range`를 호출하여 기존 region을 쪼개고, 이후 `memblock_remove_region`를 호출하여 범위를 넘는 region을 제거한다.
## 3. linear map의 시작 위치 계산
``` C
memstart_addr = round_down(memblock_start_of_DRAM(),
                           ARM64_MEMSTART_ALIGN);
```
D램의 시작 위치를 읽어온 다음, alignment를 맞춰서 시작 위치를 결정한다.

실제 Dram의 물리주소는 0부터 바로 사용할 수는 없다. 따라서, 0이 아닌 임의의 물리 주소가 매핑되어야 하는데, alignment를 지켜줘야 하므로, round down을 수행한다.

round up을 할 경우, 일부 주소를 사용할 수 없게 되어버린다.
## 4. 커널 바이너리 위치에 따른 linear map 위치 조정
커널 바이너리는 반드시 reserve로 보호해야 하므로, linear map 영역에 포함되어야만 한다.
따라서 다음과 같은 보정을 수행한다.

먼저 linear map에 넣을 수 없을 만큼 큰 메모리를 탑재한 경우, 다음과 같은 방법으로 버린다. 여기서 버릴 때, 커널 바이너리의 위치를 고려한다.
``` C
memblock_remove(max_t(u64, memstart_addr + linear_region_size,
			__pa_symbol(_end)), ULLONG_MAX); // 초과된 메모리 버림
// 초과 메모리를 버릴 때, 커널 바이너리를 지켜야 하므로, _end에 대한 max를 수행한다.
 
// 이하의 조건식은 __pa_symbol(_end)가 의미하는 커널 바이너리 때문
// 커널 바이너리를 지키기 위해서 linear_region_size보다 크게 잡힌 경우
// memstart_addr을 이동시켜서 그 크기를 linear_region_size로 맞춘다
if (memstart_addr + linear_region_size < memblock_end_of_DRAM()) {
	/* ensure that memstart_addr remains sufficiently aligned */
	memstart_addr = round_up(memblock_end_of_DRAM() - linear_region_size,
				 ARM64_MEMSTART_ALIGN);
	memblock_remove(0, memstart_addr);
}
```
`__pa_symbol(_end)`는 커널 바이너리의 끝 주소를 의미하며, 이를 지키기 위해 max값을 취하기 때문에 linear_region_size보다 크게 잡힐 수 있다. 이 경우 memstart_addr을 당겨올려서 그 크기를 맞춰준다.

이후 부트 옵션(커널 파라미터)에 의해 설정된 메모리 상한(memory_limit)이 존재한다면, 이를 반영한다. 여기서도 마찬가지로 커널 바이너리의 위치가 limit에 걸릴 경우를 고려해서 확장한다.
``` C
	/*
	 * Apply the memory limit if it was set. Since the kernel may be loaded
	 * high up in memory, add back the kernel region that must be accessible
	 * via the linear mapping.
	 */
	if (memory_limit != PHYS_ADDR_MAX) {
		// memory_limit에 의한 조정
		memblock_mem_limit_remove_map(memory_limit);
		// 조정되었음에도 커널 바이너리는 지켜야 한다, 따라서 재조정
		memblock_add(__pa_symbol(_text), (resource_size_t)(_end - _text));
	}
```
## 5. initrd 처리
initrd는 init ram disk의 약자로, 부팅 시 사용할 시스템 바이너리이다. 이는 부팅에서 필수적인 영역으로 반드시 보존해야 한다. 따라서 다음을 수행한다.
``` C
// 커널 config에 의한 initrd 사용 여부 및 size확인
if (IS_ENABLED(CONFIG_BLK_DEV_INITRD) && phys_initrd_size) {
		// 시작 위치 확인
		phys_addr_t base = phys_initrd_start & PAGE_MASK;
		// alignment를 고려한 initrd 크기
		resource_size_t size = PAGE_ALIGN(phys_initrd_start + phys_initrd_size) - base;

		// 기존 설정한 linear map에 포함되는지(커널이 접근 가능한지) 확인
		// kernel 및 initrd의 배치 설정은 부트로더의 책임
		// 확인만 하고 패스 
		if (WARN(base < memblock_start_of_DRAM() ||
			 base + size > memblock_start_of_DRAM() +
				       linear_region_size,
			"initrd not fully accessible via the linear mapping -- please check your bootloader ...\n")) {
			// linear map 영역 바깥, 커널이 접근 불가능하므로 사실상 비채 실패
			phys_initrd_size = 0;
		} else {
			// 전체 영역인 memtype memory에 initrd 부분 추가
			memblock_add(base, size);
			memblock_clear_nomap(base, size);
			// 반드시 보호해야 하는 부분이므로 memtype reserve에도 추가
			// buddy allocator등이 임의로 사용하여 덮어쓰지 않도록 방어하는 용도
			memblock_reserve(base, size);
		}
	}

```
## 6. 커널 이미지 보호
initrd에 이어서 커널 이미지도 보호한다.
``` C
	// 커널 이미지 영역을 reserve 처리
	memblock_reserve(__pa_symbol(_text), _end - _text);
	
```
## 7. memblock의 initrd 위치를 가상 주소로 변환
``` C
if (IS_ENABLED(CONFIG_BLK_DEV_INITRD) && phys_initrd_size) {
	initrd_start = __phys_to_virt(phys_initrd_start);
	initrd_end = initrd_start + phys_initrd_size;
}
```
앞서 5번에서 initrd의 linear map 영역 존재 여부를 확인한 다음, 계산한 initrd의 물리 주소와 그 size를 가상 주소로 변환, 전역 변수에 저장하여 추후 활용한다. 

이는 추후 호출될 공통 리눅스 커널 코드가 initrd의 위치를 가상주소로 기대하기 때문이다. 이 과정이 뒤늦게 수행되는 이유는 initrd가 linear map 영역 내에 들어왔음이 보장되었기 때문이다.
## 8. reserved memory 초기화
``` C
early_init_fdt_scan_reserved_mem();
```
앞서 memtype memory는 DT에 의해서 초기화가 되었고, memblock_init에서 적절히 설정이 되었다. 이제 memtype memory가 구해졌으므로, DT에 정의된 reserved memory 리스트를 처리하며 memtype reserved를 초기화한다.

해당 함수는 `drivers/of/fdt.c`에 존재하며, flatten device tree를 순회하며 reserved memory 항목에 대하여 memtype reserved에 region을 추가한다.
> of는 open firmware의 약자이다.

해당 device tree의 내용은 하드웨어에 특화된 부분이며, firmware 메모리, DMA 영역, secure world, 등이다.
# Summary
즉, 이 함수는 커널 바이너리와 initrd를 지키면서 linear map을 초기화 하고, 부트로더가 DT로 전달한 반드시 보존해야 하는 메모리 영역을 memtype reserved로 보호한다. 
# Reference
- 분석은 [여기](https://elixir.bootlin.com/linux/v6.18/source/arch/arm64/mm/init.c#L185)에서 했음.
- [전체 코드](https://github.com/torvalds/linux/blob/v6.18/arch/arm64/mm/init.c)는 다음과 같다.
``` C
// linux kernel, 6.18 
void __init arm64_memblock_init(void)
{
	s64 linear_region_size = PAGE_END - _PAGE_OFFSET(vabits_actual);

	/*
	 * Corner case: 52-bit VA capable systems running KVM in nVHE mode may
	 * be limited in their ability to support a linear map that exceeds 51
	 * bits of VA space, depending on the placement of the ID map. Given
	 * that the placement of the ID map may be randomized, let's simply
	 * limit the kernel's linear map to 51 bits as well if we detect this
	 * configuration.
	 */
	if (IS_ENABLED(CONFIG_KVM) && vabits_actual == 52 &&
	    is_hyp_mode_available() && !is_kernel_in_hyp_mode()) {
		pr_info("Capping linear region to 51 bits for KVM in nVHE mode on LVA capable hardware.\n");
		linear_region_size = min_t(u64, linear_region_size, BIT(51));
	}

	/* Remove memory above our supported physical address size */
	memblock_remove(1ULL << PHYS_MASK_SHIFT, ULLONG_MAX);

	/*
	 * Select a suitable value for the base of physical memory.
	 */
	memstart_addr = round_down(memblock_start_of_DRAM(),
				   ARM64_MEMSTART_ALIGN);

	if ((memblock_end_of_DRAM() - memstart_addr) > linear_region_size)
		pr_warn("Memory doesn't fit in the linear mapping, VA_BITS too small\n");

	/*
	 * Remove the memory that we will not be able to cover with the
	 * linear mapping. Take care not to clip the kernel which may be
	 * high in memory.
	 */
	memblock_remove(max_t(u64, memstart_addr + linear_region_size,
			__pa_symbol(_end)), ULLONG_MAX);
	if (memstart_addr + linear_region_size < memblock_end_of_DRAM()) {
		/* ensure that memstart_addr remains sufficiently aligned */
		memstart_addr = round_up(memblock_end_of_DRAM() - linear_region_size,
					 ARM64_MEMSTART_ALIGN);
		memblock_remove(0, memstart_addr);
	}

	/*
	 * If we are running with a 52-bit kernel VA config on a system that
	 * does not support it, we have to place the available physical
	 * memory in the 48-bit addressable part of the linear region, i.e.,
	 * we have to move it upward. Since memstart_addr represents the
	 * physical address of PAGE_OFFSET, we have to *subtract* from it.
	 */
	if (IS_ENABLED(CONFIG_ARM64_VA_BITS_52) && (vabits_actual != 52))
		memstart_addr -= _PAGE_OFFSET(vabits_actual) - _PAGE_OFFSET(52);

	/*
	 * Apply the memory limit if it was set. Since the kernel may be loaded
	 * high up in memory, add back the kernel region that must be accessible
	 * via the linear mapping.
	 */
	if (memory_limit != PHYS_ADDR_MAX) {
		memblock_mem_limit_remove_map(memory_limit);
		memblock_add(__pa_symbol(_text), (resource_size_t)(_end - _text));
	}

	if (IS_ENABLED(CONFIG_BLK_DEV_INITRD) && phys_initrd_size) {
		/*
		 * Add back the memory we just removed if it results in the
		 * initrd to become inaccessible via the linear mapping.
		 * Otherwise, this is a no-op
		 */
		phys_addr_t base = phys_initrd_start & PAGE_MASK;
		resource_size_t size = PAGE_ALIGN(phys_initrd_start + phys_initrd_size) - base;

		/*
		 * We can only add back the initrd memory if we don't end up
		 * with more memory than we can address via the linear mapping.
		 * It is up to the bootloader to position the kernel and the
		 * initrd reasonably close to each other (i.e., within 32 GB of
		 * each other) so that all granule/#levels combinations can
		 * always access both.
		 */
		if (WARN(base < memblock_start_of_DRAM() ||
			 base + size > memblock_start_of_DRAM() +
				       linear_region_size,
			"initrd not fully accessible via the linear mapping -- please check your bootloader ...\n")) {
			phys_initrd_size = 0;
		} else {
			memblock_add(base, size);
			memblock_clear_nomap(base, size);
			memblock_reserve(base, size);
		}
	}

	/*
	 * Register the kernel text, kernel data, initrd, and initial
	 * pagetables with memblock.
	 */
	memblock_reserve(__pa_symbol(_text), _end - _text);
	if (IS_ENABLED(CONFIG_BLK_DEV_INITRD) && phys_initrd_size) {
		/* the generic initrd code expects virtual addresses */
		initrd_start = __phys_to_virt(phys_initrd_start);
		initrd_end = initrd_start + phys_initrd_size;
	}

	early_init_fdt_scan_reserved_mem();
}
```