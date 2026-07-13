---
Date: 2026-07-07
tags:
  - Linux
  - Memory
---
# Overview
linear map 위에 초기 페이지 테이블을 초기화한다.

> [[arm64_memblock_init]]에서 이어진다.
# Contents
## 1. map_mem
여기서는 앞서 [[arm64_memblock_init]]에서 재단한 memblock의 region의 내용으로 메모리 매핑, 즉 페이지 테이블의 초기화를 수행한다.
``` c
// arm64의 페이지 테이블은 설정에 따라서 다양한 단계를 가질 수 있다
// 4KB granule(1 PTE의 매핑 크기)을 기준으로 1레벨에 9bit, 512개의 entry를 가진다
// pgd는 총 39bit 512GB, pud는 30bit 1GB, pmd는 21 bit로 2MB를 갖는다. 
// 여기서 인자로 들어오는 pgd_t는 linear map 전체를 추가할 커널 페이지 테이블의 root이다  
static void __init map_mem(pgd_t *pgdp)
{
	static const u64 direct_map_end = _PAGE_END(VA_BITS_MIN);
	phys_addr_t kernel_start = __pa_symbol(_text);
	phys_addr_t kernel_end = __pa_symbol(__init_begin);
	phys_addr_t start, end;
	phys_addr_t early_kfence_pool;
	int flags = NO_EXEC_MAPPINGS;
	u64 i;

	BUILD_BUG_ON(pgd_index(direct_map_end - 1) == pgd_index(direct_map_end) &&
		     pgd_index(_PAGE_OFFSET(VA_BITS_MIN)) != PTRS_PER_PGD - 1);

	early_kfence_pool = arm64_kfence_alloc_pool();

	linear_map_requires_bbml2 = !force_pte_mapping() && can_set_direct_map();

	if (force_pte_mapping())
		flags |= NO_BLOCK_MAPPINGS | NO_CONT_MAPPINGS;

	// vmlinux 영역의 memblock region에 대해서 nomap 설정을 함
	// nomap 설정을 통해서 추후 있을 for loop에서 초기화를 방지
	// 그 이유는 권한 세분화를 위함임
	memblock_mark_nomap(kernel_start, kernel_end - kernel_start);

	// 앞서 위치를 결정한 linear map 영역에 대한 page table을 초기화
	// 여기서 NOMAP 영역을 제외하고 초기화 수행함. 
	for_each_mem_range(i, &start, &end) {
		if (start >= end)
			break;
		
		// linear map 영역은 기본적으로 read/write 가능해야 함
		// 이는 page kernel 옵션과 동일한 속성임
		__map_memblock(pgdp, start, end, pgprot_tagged(PAGE_KERNEL),
			       flags);
	}

	 // linear map위의 vmlinux에 대한 페이지 테이블 초기화
	 // no map을 걸고 별도로 테이블 초기화를 수행하는 이유는 권한의 세분화를 위함
	 // 한번에 같이 초기화되면 최대한 큰 block으로 뭉쳐지고, 권한이 통일되어 보호 불가능
	 // symbol기반 매핑(_text 등)과 linear map은 서로 다른 매핑, 
	 // 어느 한 쪽이 보호(read only 등)를 우회시키지 말아야 한다.
	 // 물리 연속 매핑을 방지하여 권한 세분화를 준비함
	 // 현재 설정은 임시로 RW 가능, 본래 vmlinux는 보다 세부적인 권한이 필요하다
	__map_memblock(pgdp, kernel_start, kernel_end,
		       PAGE_KERNEL, NO_CONT_MAPPINGS);
			   
	// 앞서 vmlinux 위에 설정한 no map을 제거함
	memblock_clear_nomap(kernel_start, kernel_end - kernel_start);
	
	// fence pool 초기화, 현재 다루지 않음.
	arm64_kfence_map_pool(early_kfence_pool, pgdp);
}
```
이 구현에서 핵심 포인트는 vmlinux를 여타 linear map과 따로 매핑한다는 데에 있다. 이를 위해 ``
`memblock_mark_nomap`를 활용한다.  `memblock_mark_nomap`은 인자로 전달된 영역의 memblock region에 대하여 no map 플래그를 설정하며, 이 플래그가 설정된 region은 for loop에서 skip된다. 

> `for_each_mem_range` -> `__next_mem_range` -> `should_skip_region`에서 `MEMBLOCK_NOMAP`이 region을 필터링한다.

따로 매핑을 수행하는 이유는 페이지 테이블이 관리하는 **권한의 세분화**를 위함이다. 기본적인 kernel의 page는 범용한 용도이므로, **readable/writable하지만 executable해서는 안된다**. 그러나 vmlinux의 **text 영역의 경우에는 readonly & executable**해야 하고, 그 외의 **read only**등이 필요하다. 따라서 이들은 **반드시 서로 다른 entry를 가져야** 하는데, 함께 매핑을 수행할 경우, 최대한 큰 chunk를 유지하려는 성향으로 인해서 하나의 큰 엔트리로 묶이게 되며 세분화에 실패한다.

> vmlinux는 기본적으로 readable/writable한 direct map과 달리, 세세한 권한 설정이 필요하다.

따라서 `MEMBLOCK_NOMAP`으로 설정하여 for loop에서 vmlinux 영역을 제거하고, `__map_memblock`을 별도로 호출하는 것으로 별도의 테이블 엔트리를 갖도록 한다.

현재 구현에서는 vmlinux도 page kernel로 권한이 설정되는데, 이후 별도의 함수에서 vmlinux의 segment에 맞는 권한을 갖도록 수정된다. 

실제 페이지 테이블 초기화는 `__map_memblock`이 수행하며, 나중에 기회가 된다면 따로 다룰 예정이다.
## 2. memblock_allow_resize
[[memblock]]에서 region의 배열은 vector마냥 resizable하다.

그러나 해당 함수 실행 이전까지는 resize가 불가능했는데, 이는 page table이 없어 메모리 충돌이 발생할 위험이 있기 때문이었다.

그러나 앞서 map_mem을 통해 매핑을 구축하였으므로, resize가 가능하도록 flag를 활성화 한다.
## 3. create_idmap
identity mapping 영역을 구축한다. idmap이라 함은 가상 주소와 물리 주소가 같은 것을 뜻한다.

이는 매우 예외적인 매핑으로, **MMU/페이지 테이블 전환**이라는 특수한 경우에 대해서만 제한적으로 사용된다.

> 페이지 테이블 전환 관련 내용은 현재 다루지 않는다.
## 4. declare_kernel_vmas
다음은 커널 바이너리가 위치한 각 메모리 세그먼트에 대해서 vma(virtual memory address)에 사용을 선언한다. 이후 vmalloc 할당자는 이 선언을 바탕으로 커널 바이너리의 손상을 예방한다.
``` c
static void __init declare_kernel_vmas(void)
{
	static struct vm_struct vmlinux_seg[KERNEL_SEGMENT_COUNT];

	// 일반 커널 코드
	declare_vma(&vmlinux_seg[0], _text, _etext, VM_NO_GUARD);
	// 커널 바이너리의 영구적인 read only data
	declare_vma(&vmlinux_seg[1], __start_rodata, __inittext_begin, VM_NO_GUARD);
	// 초기화 전용 코드
	declare_vma(&vmlinux_seg[2], __inittext_begin, __inittext_end, VM_NO_GUARD);
	// 초기화 전용 데이터
	declare_vma(&vmlinux_seg[3], __initdata_begin, __initdata_end, VM_NO_GUARD);
	// 영구 writable data, BSS 
	declare_vma(&vmlinux_seg[4], _data, _end, 0);
}
```
`declare_vma`는 va를 바탕으로 `vm_struct`를 초기화를 수행하고, 이를 할당자 vma에 전달하여 등록한다. 

각 영역의 의미는 다음과 같다.

- \[text, \_etext\): 부팅 이후에도 계속 사용될 커널 코드, `.text` 류. 실행 가능, 쓰기 금지.
- \[\_start_rodata, \_inittext_begin): 영구 사용 읽기 전용 데이터, 리터럴, const, exception, ...
- \[\_inittext_begin, \_inittext_end): 초기화용 커널 코드, 부팅 전용이므로 나중에 정리됨
- \[\_initdata_begin, \_initdata_end): 마찬가지로 부팅용 데이터, 나중에 정리됨
- \[\_data, \_end): 영구 사용 쓰기 가능 데이터, `.data` 전역변수, 전역 자료구조, 등

`guard page`는 각 세그먼트 사이에 보호 용도로 추가하는 페이지인데, stack canary와 유사한 역할을 한다. 만약 세그먼트를 초과하는 사용이 발생할 경우, 초과분이 `guard page`로 이어져 seg fault를 유발하여 메모리 오염을 경고한다. 

그러나 앞서 선언한 4개의 세그먼트의 경우 `VM_NO_GUARD` 플래그를 설정하여 `guard page`의 생성을 막는데, 이는 앞의 4개 세그먼트는 vmlinux의 일부로서 반드시 인접해야 하기 때문이다. 따라서 vmlinux의 끝인 `_end`이후에만 `guard page`를 넣는다.

> vmlinux는 linear map에 존재하므로, 매핑은 이미 이루어졌고, 여기서는 vma에 등록만 한다.

> \_\_inittext와 \_\_initdata는 오직 부팅 용도로만 사용되며, 이후에 제거된다.
# Summary
`memblock_init`이 주어진 DRAM 중 사용할 영역을 선별했다면, `paging_init`은 그 linear map의 실체인 매핑, 즉 페이지 테이블 초기화를 수행한다.

공부와는 별개로 arm64 리눅스 부팅 코드를 읽어왔는데, 잠깐 끊고 가려고 한다. 부팅이란 결국 런타임 환경의 완성을 향하는 과정인데, 난 그 최종 완성 상태에 대해서 너무나도 무지하다. 사실상 부팅 코드를 보면서 완성된 상태를 파편적으로 습득하는데, 이는 굉장히 효율이 좋지 않다.

나중에 성장한 다음, 돌아오는 방향이 좋겠다. 특히 `bootmem_init` 등은 zone이나 buddy allocator 등의 개념이 필요하다고 생각된다.
# Reference
- 분석은 [여기](https://elixir.bootlin.com/linux/v6.18/source/arch/arm64/mm/mmu.c#L1351)에서 했음.
- [전체 코드](https://github.com/torvalds/linux/blob/v6.18/arch/arm64/mm/mmu.c)는 다음과 같다.
``` c
void __init paging_init(void)
{
	map_mem(swapper_pg_dir);

	memblock_allow_resize();

	create_idmap();
	declare_kernel_vmas();
}

```