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

> Linux 6.18 기준

해당 함수는 이미 memblock에 채워진 region에 대해 **linear map의 위치**를 설정한다. 또한 주요 보존 위치에 대해서 reserve를 수행한다. 
# Pre-requisite: linear map
linear map은 가상 메모리의 한 종류로, **물리메모리와 선형으로 연속되게 매핑**하는 영역이다. 선형으로 매핑된다 함은, offset 계산으로 매핑함을 뜻하며, 주소 변환 방식은 다음과 같다.
$$
VAlinear=PAGE\_OFFSET+(PA−memstart\_addr)
$$
즉, 변환을 위한 offset만 구하면 즉시 물리-가상 전환이 가능하다. 이러한 특성 탓에 **direct map**이라고 불리기도 한다.

**linear map은 오직 DRAM**만 취급하며, MMIO나 불연속 매핑을 위한 vmalloc, 등을 위한 가상 주소는 **linear map에서 관리하지 않는다**. 다만, vmalloc이 linear map을 다시 한 번 매핑하여 사용할 수는 있다. 

> 메모리 관련 개념은 [[물리 메모리 VS 가상 메모리]] 정리 참고
# Contents
## 0. mental model - sliding window
memblock_init에서 초기화하는 **linear map은 sliding window로 모델링** 할 수 있다. linear map 영역의 크기는 vabits에 의해 고정되어 있으며, 여러 필요에 의해 그 위치가 이동하기 때문이다.
## 1. linear map 크기 계산
가장 먼저 linear map 영역 크기를 계산한다. 계산 식은 다음과 같다.
``` c
s64 linear_region_size = PAGE_END - _PAGE_OFFSET(vabits_actual);
```
단순히 커널 빌드 옵션 등으로 설정한 끝 주소에서 시작 주소를 빼서 크기를 구한다.

먼저 **vabits_actual은 현제 CPU가 주소 표현에 사용하는 데에 쓰는 bit의 수**다. 64bit 아키텍쳐에서 포인터는 64bit이지만, 이 중에서 vabits_actual만 활용해서 RAM에서 값을 읽어오고, 나머지는 비트는 메모리의 접근 위치를 구분하는 용도로 사용한다. 이는 특정 레지스터에서 읽어서 확인한다.

> ARM64에서 vabits_actual은 보통 48이다.

\_PAGE_OFFSET() 매크로의 역할은, 정수를 받아서 해당 번째 비트 이상을 마스킹하는 값을 만드는것이다. 정의는 다음과 같다.
``` c
#define _PAGE_OFFSET(va) (-(UL(1) << (va)))
```
여기서 -를 곱한다는 것은, 모든 비트를 flip하고 1을 더하는 것이다. 따라서 val만큼 1을 shift한 다음, 모든 비트를 반전하고, 다시 1을 더한다.
``` c
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
``` c
PAGE_END == -(1 << (48 - 1)) == 0xffff_8000_0000_0000;
```
PAGE_END의 의미는 linux 커널이 정의하는 메모리 표현 범위의 한계이다. 그리고 이 값은 보통 48비트로 본다.

vabits가 보통 48이므로, 단순 계산으로는 -2^47 + 2^48이므로, linear map의 크기는 2^47이다.

이러한 방식으로 linear map 영역의 주소가 설정된 이유는 커널이 가상 주소의 특정 비트(주로 msb, 즉 63번 비트)를 보고 해당 주소가 어떤 위치(유저, 커널, 커널의 특정 영역, 등)의 메모리인지 구분하기 때문이다.

> 즉, linear map을 매핑한 window의 size를 계산한다.
## 2. 물리 주소 초과분 제거거
CPU가 물리 주소로는 사용할 수 없는 만큼의 ram을 탑재한 경우(physical address bit를 초과), 초과분을 포기한다. memblock에서는 다음과 같이 제거된다.
``` c
memblock_remove(1ULL << PHYS_MASK_SHIFT, ULLONG_MAX);
```
여기서 `PHYS_MASK_SHIFT`는 유효한 물리 주소 범위(하위 비트 수)를 의미하며, 유효 범위 이상의 ULLong의 최댓값만큼(즉 모두)을 memblock에서 제외함을 뜻한다.

`memblock_remove`함수는 내부적으로 `memblock_remove_rnage`함수를 호출하며, `memblock_isolate_range`를 호출하여 기존 region을 쪼개고, 이후 `memblock_remove_region`를 호출하여 범위를 넘는 region을 제거한다.

> 앞서 계산한 window size를 바탕으로 window의 끝을 결정한다.
## 3. linear map의 시작 위치 계산
``` c
memstart_addr = round_down(memblock_start_of_DRAM(),
                           ARM64_MEMSTART_ALIGN);
```
D램의 시작 위치를 읽어온 다음, alignment를 맞춰서 시작 위치를 결정한다.

실제 Dram의 유효한 물리주소는 여러 이유로 다양할 수 있다(0이라는 보장이 없다). 따라서 linear map 영역에 매핑할 유효한 물리 주소의 시작 위치를 읽어줘야 한다(부트로더가 읽어왔음).

여기서 이후에 있을 large block mapping의 편의과 memory alignment를 지키기 위해 round downd을 수행한다.

round down을 할 경우, 일부 가상 주소에는 유효한 ram이 할당되지 않고, 따라서 가상 주소가 낭비되는 문제가 있다. 그러나 역으로 round up을 할 경우, **유효한 물리 주소를 버리게** 되어버린다.

> window의 시작을 결정한다.
## 4. 커널 바이너리 위치에 따른 linear map 위치 조정
커널 바이너리는 반드시 linear map으로 접근해야 하므로, linear map 영역에 포함되어야만 한다.
따라서 다음과 같은 보정을 수행한다.

먼저 linear map에 넣을 수 없을 만큼 큰 메모리를 탑재한 경우, 다음과 같은 방법으로 버린다. 여기서 버릴 때, 커널 바이너리의 위치를 고려한다.
``` c
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
`__pa_symbol(_end)`는 커널 바이너리 끝의 물리 주소를 의미하며, 이를 지키기 위해 max값을 취하기 때문에 linear_region_size보다 크게 잡힐 수 있다. 이 경우 linear map에 매핑되는 물리 시작 주소(memstart_addr)를 끌어올려서 그 위치를 맞춰준다.

이는 커널 바이너리를 보호하기 위한 조치로, memstart가 위로 조정되기에 일부 low한 RAM이 버려지는 효과가 있다.

> linear map이 kernel을 반드시 포함해야 하므로, window의 위치를 이동한다.

## 5. 리눅스 커널에 의한 메모리 초과분 정리
이후 부트 옵션(커널 파라미터)에 의해 설정된 메모리 상한(memory_limit)이 존재한다면, 이를 반영한다. 여기서도 마찬가지로 커널 바이너리의 위치가 limit에 걸릴 경우를 고려해서 확장한다.
``` c
	/*
	 * Apply the memory limit if it was set. Since the kernel may be loaded
	 * high up in memory, add back the kernel region that must be accessible
	 * via the linear mapping.
	 */
	if (memory_limit != PHYS_ADDR_MAX) {
		// memory_limit에 의한 조정
		memblock_mem_limit_remove_map(memory_limit);
		// 조정되었음에도 커널 바이너리는 지켜야 한다, 따라서 다시 추가
		memblock_add(__pa_symbol(_text), (resource_size_t)(_end - _text));
	}
```
여기서도 커널 바이너리는 반드시 지켜져야 하므로, 다시 add하여 추가한다.
## 6. initrd 처리
initrd는 init ram disk의 약자로, 부팅 시 사용할 시스템 바이너리이다. 이는 부팅 과정에서 필요한 영역으로 보존해야 한다. 따라서 다음을 수행한다.
``` c
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
			// linear map 영역 바깥, 커널이 접근 불가능하므로 사실상 배치 실패
			// 사용 포기
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
linear map 영역 안에 있다면 추가 및 reserve로 보호하지만, linear map 외부에 걸린다면 사용을 포기해버린다. 따라서 initrd 사용이 필요하다면 kernel 이미지 근처에 initrd를 배치해줘야 한다.

> kernel과 initrd는 32G 이내에 배치되어야 정상적으로 인식된다.

> 영역 내부라면 initrd 부분 추가, 바깥에 걸린다면 사용 포기.
## 7. 커널 이미지 보호
initrd에 이어서 커널 이미지도 보호한다.
``` c
	// 커널 이미지 영역을 reserve 처리
	memblock_reserve(__pa_symbol(_text), _end - _text);
```
## 8. memblock의 initrd 위치를 가상 주소로 변환
``` C
if (IS_ENABLED(CONFIG_BLK_DEV_INITRD) && phys_initrd_size) {
	initrd_start = __phys_to_virt(phys_initrd_start);
	initrd_end = initrd_start + phys_initrd_size;
}
```
앞서 5번에서 initrd의 linear map 영역 존재 여부를 확인한 다음, 계산한 initrd의 물리 주소와 그 size를 가상 주소로 변환, 전역 변수에 저장하여 추후 활용한다. 

이는 추후 호출될 공통 리눅스 커널 코드가 initrd의 위치를 가상주소로 기대하기 때문이다. 이 과정이 뒤늦게 수행되는 이유는 initrd가 linear map 영역 내에 들어왔음이 보장되었기 때문이다.
## 9. reserved memory 초기화
``` C
early_init_fdt_scan_reserved_mem();
```
앞서 memtype memory는 DT에 의해서 초기화가 되었고, memblock_init에서 적절히 설정이 되었다. 이제 memtype memory가 구해졌으므로, DT에 정의된 reserved memory 리스트를 처리하며 memtype reserved를 초기화한다.

해당 함수는 `drivers/of/fdt.c`에 존재하며, flatten device tree를 순회하며 reserved memory 항목에 대하여 memtype reserved에 region을 추가한다.
> of는 open firmware의 약자이다.

해당 device tree의 내용은 하드웨어에 특화된 부분이며, firmware 메모리, DMA 영역, secure world, 등이다.

reserved는 해당 영역을 allocator에 제공하지 않는다는 의미이지 여전히 linear map의 일부이다.
# Summary
즉, 이 함수는 커널 바이너리와 initrd를 지키면서 linear map을 초기화 하고, 부트로더가 DT로 전달한 반드시 보존해야 하는 메모리 영역을 memtype reserved로 보호한다. 

이 linear map 영역은 일종의 고정된 size의 window이며, size는 vabits로 결정되고, window의 위치는 사용 가능한 DRAM 중, kernel과 initrd를 반드시 덮는 위치로 한다.
# Reference
- 분석은 [여기](https://elixir.bootlin.com/linux/v6.18/source/arch/arm64/mm/init.c#L185)에서 했음.
- [전체 코드](https://github.com/torvalds/linux/blob/v6.18/arch/arm64/mm/init.c)는 다음과 같다.
``` c
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