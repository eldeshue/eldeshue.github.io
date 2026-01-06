---
Date: 2025-06-30
tags:
  - Graphics
  - DirectX12
  - DirectX
---
# Overview
GPU의 메모리와 연결해서 정리해보는 DX12의 Descriptor 
# Resources - Texture
일반적으로 DX12에서 말하는 대부분의 자원은 texture의 형태로 저장된다. 

texture는 **임의의 tuple로 구성된 다차원(주로 1, 2, 3차) 배열**로, 필요와 목적에 따라 그 tuple의 구성이 달라진다. [[DXGI]]에서 정의하는 texture의 타입을 다음과 같은 형태로 구성되는데, 일부 예외가 있을 수 있다.
``` C++
// 다음은 DXGI에서 정의하는 텍스쳐 자원의 타입에 대한 형식이다.
DXGI_FORMAT_TEXEL_TYPE

// 텍셀은 다음과 같이 표현된다.
// 예를 들어 픽셀 혹은 법선 벡터를 표현하는 float4의 경우
// R, G, B, A 각각이 32비트 float이다.
DXGI_FORMAT_R32G32B32A32_FLOAT

// zbuffer에 사용하는 [0, 1]로 사상되는 부호 있는 16bit 버퍼의 경우
// Depth 16, 
DXGI_FORMAT_D16_UNORM
```
이들 텍스쳐는 일반적으로 힙에 저장되며, 렌더링의 결과(view buffer)가 되어 다시 mesh에 씌워지거나(texture), 깊이 값을 판별하는 용도(depth buffer), 혹은 stencil buffer가 되어 렌더링의 전처리 및 후처리 등 다양한 방향으로 사용될 수 있다.
# Descriptor
descriptor란 렌더링에 개입하는 자원을 서술하는 view이며, GPU가 리소스를 접근하기 위한 **메타데이터**다.

>  **디스크립터는 GPU의 가상 메모리(Virtual Address Space)를 간접적으로 다루기 위한 '메타데이터 핸들러'.**  

> 즉, **"이 리소스는 어디에 있고, 어떤 포맷이고, 어떻게 샘플링하거나 접근해야 하는가?"**를 정의하는 구조체다.

각 디스크립터는 다음과 같은 리소스를 참조할 수 있다:
- **CBV (Constant Buffer View)**
- **SRV (Shader Resource View)** — 텍스처, 버퍼
- **UAV (Unordered Access View)** — RW 버퍼, RW 텍스처
- **Sampler**
- **RTV (Render Target View)**
- **DSV (Depth Stencil View)**
## 왜 DX12는 디스크립터를 필요로 하는가?
그렇다면, DX12는 왜 자원을 직접 접근하지 않고, 굳이 desc를 통해서 접근하는가? 

현대 gpu의 virtual address로 인해, gpu 자원은 기본적으로 메인 메모리의 한 부분에 불과하다. 즉, **자원은 범용적이다.** 이는 달리 말하면, 해당 메모리 영역은 gpu가 어떻게 읽어야 하는지 모른다는 의미이다. 따라서 우리는 **gpu에게 해당 자원을 어떻게 사용할 것인지에 대해서 알려야 하며,** 이 메타정보가 곧 descriptor가 된다.

> **즉, gpu는 desc의 정보는 읽을 수 있고, desc의 내용을 바탕으로 ram의 자원을 해석한다.**
# GPU Virtual Address
그렇다면, gpu는 어떻게 ram을 읽을 수 있을까? 그것은 현대 gpu가 virtual address로 인해서 메인 메모리를 매핑하기 때문이다.

현대 gpu는 거의 독립된 별도의 컴퓨터다. GPU는 독자적인 **virtual memory management를 수행하며, 별도의 virtual address를 가질 수 있다.** 이러한 맥락에서 gpu가 desc를 통해서 ram의 정보에 접근하면, IOMMU는 DMA를 수행, RAM에 위치하는 실제 데이터를 읽어온다.

> **즉, gpu의 입장에서 main memory는 swap space를 저장하는 secondary memory와 같다.**
> > 실제로 Evict 혹은 Resident api를 사용하여 제어할 수 있음.

따라서, binding된 자원은 명시적으로 vram에 적재되는 것이 아니라, rendering pipeline의 수행 과정에서 해당 자원이 필요해지는 순간 적재되거나 혹은 캐싱 된 데이터를 재사용한다. 즉 gpu의 자원은, demand loading된다. 
### Gpu의 자원 참조 방식 - descriptor table
shader 코드가 descriptor를 통해서 자원을 참조하는 과정은 다음과 같다.
``` text
Shader 코드
  ↓
디스크립터 테이블 (테이블에 있는 디스크립터에 인덱싱)
  ↓
[Descriptor] 
   ↓ (GPU Virtual Address + 포맷/옵션 등 메타데이터)
[GPU Virtual Memory (VA)]
   ↓ (GPU MMU / GMMU가 변환)
[GPU Physical Memory Address]
   ↓
[VRAM (또는 RAM, 또는 기타)]
```
셰이더를 실행하는 과정에서 bind 된 자원을 참조하는 순간이 오면, 실제 디스크립터의 위치를 매핑하는 descriptor table 을 참조하여 디스크립터를 획득하고, 획득한 디스크립터를 바탕으로 실제 자원에 접근한다.
## 디스크립터 테이블과 페이지 테이블
gpu가 디스크립터를 통해서 자원을 다루는 방식은 OS에서 page의 형태로 physical memory를 관리하는 방식과 매우 닮아있다. 

cpu는 메모리에 접근하기 위해서 페이지 테이블을 확인하여 원하는 메모리 주소에 해당하는 페이지를 찾는다. 만약 해당 페이지가 존재하지 않는다면, 메인 메모리로 해당 페이지를 load한다. 이후 해당 페이지에 접근하여 참조를 수행한다.

gpu는 자원을 참조하기 위해서 디스크립터 테이블을 확인하여 디스크립터를 획득한다. 이후 디스크립터를 통해서 실제 자원을 참조한다. 이 과정에서 자원이 vram에 존재하지 않는다면, 이를 vram으로 load한다.
# Descriptor Heap
gpu가 참조할 자원은 DirectX context가 관리하는 heap에 할당된다. 이 자원을 가리키는 descriptor 또한 heap에 저장된다. 이들을 각각 resource heap, descriptor heap이라 부른다. 

이들 디스크립터는 종류에 따라서 별도의 힙에 저장되어야 한다. 그 이유는 디스크립터가 힙 내부에 연속적인 선형으로 존재하여 offset을 통해서 그 위치를 탐색하기 때문으로, 서로 다른 종류의 디스크립터를 하나의 힙에 넣으면 UB를 초래하게 된다. 
# API
## Descriptor

## Descriptor heap

### Descriptor Heap Desc




