---
Date: 2025-07-29
tags:
  - DirectX12
---
# Overview
GPU의 거동으로 알아보는 자원과 이를 저장하는 Heap.

오늘은 서로 다른 힙을 생성하는 이유와, 그 동작의 이유를 GPU의 거동과 함께 알아보자.
# GPU Resource의 다형성
모든 자원은 [[Resource|ID3D12Resource]]라는 COM 객체임. 즉, 자원의 실제 데이터(buffer) 그 자체는  동일한 format을 가지고 있음. 그러나, 이들은 용도에 따라서 서로 다른 힙에 저장됨.

이들은 그 목적에 따라 다양한 대역폭과 접근 권한으로 특수화되며, 그 결과는 대략 다음과 같다.

---
## Default Heap
default heap은 기본적으로 vram에 저장되는 것을 보장하는 heap이다. 따라서, 자주 참조하는, 하지만 변조되지 않는 데이터의 경우, default heap으로 설정하는 것이 좋다.

> **Default Heap의 데이터는 명시적으로 VRAM에 존재한다.**

이는 default heap이 gpu내부의 고속 메모리에 위치하기 때문으로, 접근 속도가 대단히 빠른 대신, CPU에서 접근할 수 없도록 되어있다.

> **Default Heap은 VRAM에 존재하는 관계로, CPU는 변조할 수 없다**

따라서, 이들을 초기화 하기 위해서는 CPU-GPU상호작용이 가능한 upload heap을 거쳐서 복사할 필요가 있다.

이들은 CPU에 의한 update가 발생되지 않으며, pipeline에서 빈번하게 참조되는 대상에 어울리므로, 다음과 같은 버퍼에 적합하다.

- Vertex buffer
- Index buffer
- Texture(너무 크면 vram에 포화를 일으키므로, 적절한 size)
- light map
- etc
## Upload Heap
GPU가 읽을 수 있는 메인 메모리의 heap. 기본적으로 CPU의 통제 하에 있는 시스템 메모리에 위치하자만, GPU의 MMU가 해당 자원 영역을 매핑하여 접근할 수 있는 메모리 영역.

> **Upload Heap은 기본적으로 시스템 메모리에 존재하나, 매핑되어 GPU가 읽을 수 있다.**

CPU와 GPU는 PCIe라는 유선 통신 방식으로 duplex 소통하는데, 이 때 DMA를 통해서 데이터가 이동해야 한다.  Upload Heap은 본질적으로 시스템 메모리에 있지만, GPU에서 접근할 수 있는데, 이는 GPU의 MMU가 해당 영역을 알고 있기 때문이다. 이 때, vram에 적재되지는 않지만, caching되어 gpu의 레지스터나 캐시에 존재할 수 있다. 

CPU입장에서 upload heap은 write 속도는 빠르지만, read 속도는 느린데, 이는 읽기 위해서 자원 상태 전환을 위한 동기화가 필요하기 때문이다.

> **CPU입장에서 Upload Heap은 write는 빠르지만, read는 매우 느리다.**

default heap에 적재해야 할 데이터의 경우, DMA를 위해서 upload heap을 거쳐간다.  이 때, 중간 버퍼가 되는 upload heap의 경우 적절한 시점에서 해제해야 메모리를 절약할 수 있다. 하지만, 언제 default heap에 적재가 완료되는지는 동기화가 필요하다. 그리고 fence를 사용한 평범한 동기화로는 성능 매우 느림. 

> **비동기적으로 동작하는 자원 upload manager가 매우 필수적임**

따라서 upload manager를 구현하여 비동기적으로 upload heap을 해제하는 테크닉이 필요하다.
### Example - Constant Buffer
아주 빈번하게 update가 발생하는 데이터가 있다. 대표적인 것이 camera와 연동된 mvp 행렬이다. 이 경우에는 데이터를 constant buffer에 넣어서 사용한다. 이러한 constant buffer에 사용하는 heap이 또 upload heap이다. 

그러나, 최근에는 다른 방식도 사용되는데, 그 중 하나가 바로 IOMMU, 즉 GPU의 virtual memory management의 일환으로, system memory를 vram에 매핑하는 것이다.

> **작은, 빈번한 업데이트가 발생하는 데이터를 위해서는 gpu virtual address mapping이 유용하다.**
## GPU Upload Heap - Resizable BAR, MMIO
Base Address Register, gpu 메모리 전체를 메인 메모리에 매핑하여, write combine을 통해서 gpu에 몰아서 update하는 방식. 즉, vram 전체에 대한 Memory Mapped IO.

> **VRAM에 위치, 메모리 매핑으로 CPU에서 접근 가능한 Heap.**

기존의 Bar는 매핑가능한 공간의 최대 크기가 256MB여서 너무 작았음. 이 size 제한을 돌파한 것이 바로 Resizable BAR임.

즉, RBAR의 도입으로 CPU가 직접 Vram에 빠른 속도로 Write 가능. 성능상의 이슈로 Read는 권장되지 않음. 기존의 upload heap에 대한 데이터의 복사가 사라지고, api호출이 간결해짐.

> **DMA를 위해서 시스템 메모리에 upload buffer를 준비하여 GPU로 하여금 읽도록 하지 말고, memory map된 GPU 메모리에 CPU가 직접 write힌다.**

이러한 CPU-GPU 사이의 address space 공유 특성은 CPU/GPU가 통합되는 **UMA 아키텍쳐**에서만 가능했음. 이미 address space를 공유하기 때문에, 불필요하게 upload buffer를 준비하고, 데이터를 복사하면서 DMA를 준비할 필요가 없음. 그러나, NUMA 아키텍쳐에서는 VRAM의 어드레스와 메인 어드레스가 서로 호환되지 않아서 현실적으로 매우 어려웠음. 

대부분의 경우 성능 향상(10% ~ 30%)을 가져오기 때문에 권장할만한 구현임. 하지만 아직 이 MMIO 기능이 구현이 안된 하드웨어가 상당함. Nvidia 기준으로 RTX3000번대 이상 GPU에서만 지원함.

더 자세한 내용은 [[Resizable BAR & GPU_UPLOAD_HEAP]]에서 확인할 수 있음.

---

앞서 제시한 예시를 바탕으로 우리는 다음의 결과를 얻을 수 있다.
> **결론 : 같은 ID3DResource 타입의 resource heap이라 해도, 그 역할에 따라서 구현은 달라짐.**
# 구현
## Heap 생성 방법
``` C++
	// Create the actual default buffer resource.
	ThrowIfFailed(device->CreateCommittedResource(
		&CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),// 디폴트 버퍼 타입으로 생성
		D3D12_HEAP_FLAG_NONE,
		&CD3DX12_RESOURCE_DESC::Buffer(byteSize),
		D3D12_RESOURCE_STATE_COMMON, // heap의 상태, 필요에 따라 상태 전환 필요
		nullptr,
		IID_PPV_ARGS(defaultBuffer.GetAddressOf())));

```
위와 같이 복잡한 인자를 전달하여 heap을 생성함. 이 buffer의 실재 데이터 및 allocating 특성 등은 여려 인자의 조합을 통해서 결정됨.

이렇게 복잡한 방식으로 초기화 하는 이유는 여러 자원이 `ID3D12Resource`타입을 공유하기 때문으로, 실제 자원의 특성에 따라 GPU의 거동이 달라져야 하기 때문임.
## D3D12_HEAP_PROPERTIES
``` C++
typedef struct D3D12_HEAP_PROPERTIES { 
	D3D12_HEAP_TYPE Type; 
	D3D12_CPU_PAGE_PROPERTY CPUPageProperty; 
	D3D12_MEMORY_POOL MemoryPoolPreference; 
	UINT CreationNodeMask; 
	UINT VisibleNodeMask; 
} D3D12_HEAP_PROPERTIES;
```
저장하려는 데이터의 특성에 맞는 힙을 생성하기 위해서 설정해야 할 값들. 설정 값 중 일부는 서로 상충하기에, 대부분의 경우에는 정해진 조합이 있다.

이러한 구조체의 초기화를 돕기 위해서 `CD3DX12_HEAP_PROPERTIES`가 있으며, 일반적으로 C가 붙은 helper쪽을 주로 사용함. 이 C가 붙은 친구는 파생 타입이라, 쉽게 생성하고, 단순 casting으로 전달 가능.
### D3D12_HEAP_TYPE
생성하려는 힙의 타입 정보를 지정하는 enum. 일반적으로 gpu 및 cpu의 접근 빈도를 기준으로 결정됨. 즉, 저장하려는 데이터의 속성에 맞는 type을 설정해야 함. 이러한 접근 빈도를 **대역폭**으로 표현.
``` C++
typedef enum D3D12_HEAP_TYPE { 
	D3D12_HEAP_TYPE_DEFAULT = 1, 
	D3D12_HEAP_TYPE_UPLOAD = 2, 
	D3D12_HEAP_TYPE_READBACK = 3, 
	D3D12_HEAP_TYPE_CUSTOM = 4, 
	D3D12_HEAP_TYPE_GPU_UPLOAD 
} ;
```
- Default 힙 : GPU가 조작하는 데이터. CPU는 제어할 수 없음. 대부분의 resource를 저장함.
- Upload 힙 : GPU로 업로드 될 데이터. cpu와 gpu가 번갈아 접근한다. 렌더링 과정에서 동적으로 업데이트 되는 데이터를 저장한다. **업로드 비용이 발생하므로 자주 업로드 할 데이터는 적절하지 않음**
- Read back 힙 : gpu가 쓰고, cpu가 읽기 위한 데이터를 저장하는 힙. sub-path의 결과를 저장함.
- custom : 사용자가 gpu와 cpu에 할당할 대역폭을 직접 지정 가능한 힙. 다만, 하드웨어를 이해해야 하기에 매우 특수한 목적으로 사용해야 함.

타입을 기준으로 한 heap properties의 기본적인 조합은 다음과 같음.
- Default : PageProperty = NOT(CPU 접근 불가능), PoolPreference = L1(GPU)
- Upload : write combine, L0
- ReadBack : write back, L0
- GPU Upload : write combine, L1

여기서 UMA 아키텍쳐인 경우, 모든 pool preference를 L0로 설정할 수 있음.

HeapType으로 `CD3D12_HEAP_PROPERTIES`를 초기화 할 수 있음.
### D3D12_CPU_PAGE_PROPERTY
생성될 힙의 page의 속성을 설정한다. 
- unknown
- CPU 접근 유무 : 접근 불가능이라면 GPU 전용을 의미한다.
- Readable/Writable 
- WRITE_BACK : 일반적인 캐시 정책, 수정 사항을 모아서 실제 READ가 발생할 때에 한 번에 반영함. 일반적으로 적당한 성능을 제공. Cache Coherency를 보장함.
- WRITE_COMBINE : WC buffer에 데이터를 모은 후, 일괄 WRITE 수행. 연속된 범위에 write는 매우 빠르지만, 읽기는 느림(CPU 입장에서 다시 읽을 일이 없어야 함), cache coherency 보장 안됨. **gpu upload heap**에 주로 설정함.
### D3D12_MEMORY_POOL
cpu의 메모리 접근 아키텍쳐(UMA / NUMA)에 따른 메모리 접근 방식을 지정. 일반적으로는 Unknown으로 설정하는게 좋아보임.
 - L0 : System Memory, 즉 NUMA
 - L1: GPU Memory, 즉 UMA
여기서 UMA 아키텍쳐인 경우, 모든 pool preference를 L0로 설정할 수 있음. 둘의 구분이 없기 때문임.
# References
- [D3D12_HEAP_PROPERTIES](https://learn.microsoft.com/ko-kr/windows/win32/api/d3d12/ns-d3d12-d3d12_heap_properties)
- CPU와 GPU 사이의 상호작용, Upload Buffer관련 영상 1 : https://www.youtube.com/watch?v=FiqoYo5S4PI&list=PL00yTT-RECdWsBjP-rQcDBelgehOyToy3&index=65