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
페이지 테이블 초기화를 실행한다.
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
# Summary
`memblock_init`이 주어진 DRAM 중 사용할 영역을 선별했다면, `paging_init`은 그 linear map을 운영하기 위한 가상 메모리를 준비한다.
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