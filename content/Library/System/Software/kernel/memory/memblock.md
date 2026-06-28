---
Date: 2026-06-17
tags:
---
# Overview
부팅 과정 초기에 사용되는 원시 메모리 할당자이며 동시에 커널의 mm모듈(buddy allocator, slab allocator, etc)를 초기화하기 위한 descriptor인 memblock에 대해서 다룬다.
# Contents
## Definition
### memblock
memblock의 정의는 다음과 같다
``` C
/**
 * struct memblock - memblock allocator metadata
 * @bottom_up: 메모리의 검색 방향이 bottom -> up 방식인지
 * @current_limit: 현재 할당 한계의 물리 주소, 현재인 이유는 ram이 추가 식별되면 늘어남
 * @memory: 시스템이 이해한 메모리 영역 전체
 * @reserved: 전체 메모리 중 예약된 영역, 커널 이미지, init ramfs 등
 */
struct memblock {
	bool bottom_up;
	phys_addr_t current_limit;
	struct memblock_type memory;
	struct memblock_type reserved;
};

extern struct memblock memblock; // 전역변수로 존재한다.
```
`bottom_up`은 `memblock`이 제공하는 메모리 할당에서 빈 메모리 영역을 탐색하는 방향을 결정한다. `bottom_up`이 `true`인 경우, 빈 공간을 아래에서 위로 탐색한다.

`memblock`의 메모리 할당 매커니즘은 기본적으로 top-down 방식인데, 이는 kernel image에 가까운 bottom 메모리 주소를 아끼기 위함이다. 보통 kernel에 가까운 주소는 각종 제약에서 자유로운 경우가 많아서 더 가치가 높다.

`current_limit`은 현재 시스템이 확보한 가용한 메모리 영역의 상한의 주소값을 의미한다. currnet인 이유는 부팅 과정에서 시스템의 메모리 이해가 진행되면 시스템이 운용할 수 있는 메모리 영역 자체가 증가하기 때문이다.

> 즉, current_limit은 현재까지 알려진 메모리 점위 중, 가용한 영역

`memblock`의 핵심역할은 메모리 상태 관리다. 이 부분을 표현하는 핵심 필드가 `memory`와 `reserved`이다. memblock은 **현재 시스템의 전체 메모리를 `memory`로 표현**하고, **사용중인 메모리만 `reserved`로** 표현한다.

> `reserved`에는 보통 kerne image, initrd, 원시 page table 등이 해당한다.

따라서 할당은 `reserved`를 제외하고 `current_limit`을 넘지 않는 범위에서 `memory`를 탐색하는 형태가 될 것이다. 
### memblock_type
``` C
/**
 * struct memblock_type - collection of memory regions of certain type
 * @cnt: number of regions, 리전의 수
 * @max: capacity of the `regions` allocated array
 * @total_size: size of all regions, 전체 할당된 byte size
 * @regions: array of regions, memblock_regions가 할당의 단위, 그 동적 배열 포인터
 * @name: the memory type
 */
struct memblock_type {
	unsigned long cnt;
	unsigned long max;
	phys_addr_t total_size;
	struct memblock_region *regions;
	char *name;
};
```
`memblock_type`은 **메모리 블록의 메타데이터**를 표현하는 구조체이다. `memblock`할당자가 다루는 메모리는 `memblock_region`단위로 할당되며, 해당 블록(전체/예약됨)의 구성을 표현한다.

`memblock_type`는 동적 할당된 배열이다. `regions`가 `memblock_region`의 배열을 가리키는 포인터이이고, `cnt`는 region의 수, `max`는 `regions`가 가리키는 배열의 크기, capacity이다. 

부팅 진행 과정에서 reserve가 늘어나거나 혹은 시스템에서 메모리를 추가로 얻거나 하면 region이 추가로 초기화되고, `regions`는 부족할 경우 reallocation된다. 또한 할당의 편리함을 위해서 `regions`가 가리키는 배열은 시작 주소를 기준으로 정렬된 상태를 유지한다.

따라서 region의 삽입/삭제는 memmove를 이용해 구현한다.

각 region은 flag로 표현되는 **속성**으로 구분되며, 그 크기가 서로 다르다. 따라서 `total_size`는 각 영역의 byte크기의 sum이다. 
### memblock_region
``` C
/**
 * struct memblock_region - represents a memory region
 * @base: region의 시작 주소, 물리주소(변환 필요)
 * @size: region의 크기
 * @flags: 해당 region의 성격(attr)을 의미하는 enum
 * @nid: NUMA node id
 */
struct memblock_region {
	phys_addr_t base;
	phys_addr_t size;
	enum memblock_flags flags;
#ifdef CONFIG_NUMA
	int nid;
#endif
};
```
할당된 각 영역을 나타내는 구조체이다. `memblock_type`에서 배열로 관리된다. 모여서 블럭을 이룬다.

각 region을 구분하는 기준은 해당 영역의 메모리가 갖는 **속성**이다. 한 영역은 반드시 이 속성을 공유해야 마땅하며, 이 속성은 `flags`로 표현된다.

특별하게도 NUMA 아키텍쳐인 경우, 해당 메모리 region이 어떤 node에 소속되었는지 식별하는 값을 가지며, 이를 `nid`로 표현한다. 이는 NUMA에 의해 특정 cpu에 locality를 존중한 메모리 우선권을 주기 위함으로, 추후 정식 allocator 초기화에 필요하다.
## Method
### memblock_isolate_range
특정 타입의 region들에 대하여 range(시작/끝 주소)체크를 수행하고, 정확히 fit하지 않은 region은 쪼개서 insert한다. 
# Summary
## 계약서
> **memblock은 계약서다.** 

리눅스 커널의 메모리 할당자는 공통 구현이다. 하지만, mm이 초기화 되지 않은 시점의 메모리 상태는 너무나도 arch-dependant하다. 최초의 메모리에 저장되는 값들은 보통 kernel image 및 initrd/initramfs인데, 이들은 bootloader가 ram에 복사하기 때문이다.

따라서 linux kernel이 제공하는 공통 메모리 allocator(buddy, slab, 등)로 이주하기 위해서는 bootloader등이 초기화 한 메모리의 상태를 정확하게 추적해야 하며, 그 역할을 수행하는 것이 바로 `memblock`이다.

`memblock`은 단순한 원시 할당자가 아니라 진짜 할당자를 초기화 하기 위한 일종의 descriptor이며, 메모리의 각 영역별 상태와 크기 및 위치를 모조리 알고 있어야 한다.

## Not a free-list
`memblock`은 메모리 상태를 그대로 표현, 유지하는 구현이다. 따라서 `memory`는 free-list가 아닌 시스템의 가용 가능한 메모리 전체를 표현한다.

만약 free-list로 구현할 경우 할당은 빠르겠지만, 메모리 상태로 복원이 어려웠으리라. 그리고 부팅 과정은 그다지 동적 할당이 자주 발생하는 구간이 아니다.
# Reference