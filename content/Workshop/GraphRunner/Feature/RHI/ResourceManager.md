---
Date: 2025-11-14
tags:
publish: "true"
---
# Overview
GraphRunner 프로젝트의 Resource와 이를 관리할 Resource Manager에 대해서 정리한다.
# Content
## Resource Manager
### Resource Factory
모든 resource는 이 resource manager를 factory로 하여 생성된다.

자원의 종류가 추가되면 resource 생성 method를 factory에 추가한다.
### Descriptor Heap Managing
mesh 관련 자원을 제외한 모두(texture, 등)을 bindless하게 운영한다.

따라서 descriptor heap과 그 heap에 해당 자원이 위치하는 index 및 stride, cnt를 관리해야 한다.

특히 heap의 index가 중요한데, 이 index를 적당히 비어있는 곳에 allocation 해줘야 할 것으로 생각된다.
### Resource Deletion
clean-up thread와 in-flight frame에 기반한 deletion queue

cpu-gpu 동기화에 의해서 자원의 삭제를 안전하게 자동화 한다.

### Resource Staging
자원 중에는 즉시 유효하지 않은 데이터가 있다. 

device local한 texture 등이 대표적으로 여기에 해당하는데, host visible에서의 복사가 필요하다.

따라서 texture 객체를 생성한다고 할 때, 이 객체는 즉시 생성되지 않으며, gpu에 의한 복사 종료 시점으로 동기화가 필요하다.

따라서 host visible한, 거대 staging buffer를 두고, 객체 생성 시점에서 gpu로의 copy를 command buffer에 record해줘야 한다.

gpu 자원은 상태를 가지는데, 이 상태의 전환도 동기화가 필요한 부분.

===

현재 능력이 부족하므로, streaming이나 multi-thread 기반의 비동기 transfer queue 사용은 어려움

비동기 transfer queue도 queue 사이의 ownership transfer를 구현해야 해서 연구가 굉장히 필요함. 이 부분은 나중에 추가하는게 이롭게 느껴짐.

따라서 pre-render upload 패턴을 채용, 단일 그래픽 큐에서 렌더링 명령 기록 시 copy도 같이(렌더링 수행 이전에) 수행하기

프레임 당 32MB 이하만 동적으로 upload하는 것을 목표로 한다.

===

자원을 생성하거나, 기존 자원에 update가 필요한 시점이 오면, 

현재 frame에 해당하는 staging buffer에 data를 memcpy하고, copy가 필요함을 자원을 나타내는 객체에 저장한다.

해당 객체를 render할 순간이 오면, 객체에 upload가 필요한지 여부를 확인하고,
필요하다면 copy 명령 기록과 barrier 삽입을 수행한다.

만약 staging buffer의 크기 부족으로 인해서 copy를 못하게 되는 경우, texture라면 default texture를 대신 전달(bindless이므로, default texture의 인덱스를 대신 전달)한다.


### Resource Residence
메모리 예산(vma 기반) 추적에 기반한 자원의 상주성 관리.

> 구현 난이도가 매우 높을 예정, 추후 구현한다.


## Resource Architecture

### VResource
혹시 모를 경우를 위한 그냥 공통 인터페이스
### VBuffer
벌칸 버퍼 객체, VkBuffer와 VmaAllocation 소유
### VImage
벌칸 이미지 객체, VkImage와 VmaImage 소유
### VImageView
Texture, etc...

실제 용도를 가지고 바인딩되는 대상

VImage에 대한 shared ptr을 소유하고 있음. -> 소멸 시점 관리

이걸 상속해서 Texture2D, Texture3D 등 구체적인 타입을 정의함.

### VBufferView

