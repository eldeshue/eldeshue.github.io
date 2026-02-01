---
Date: 2025-09-07
title: Feature
publish: "true"
---
# Feature
구현된 기능에 대한 간략한 overview.
## Render Hardware Interface, RHI
그래픽 api를 추상화 한 layer, 객체 지향적으로 구성되어야 함.

기본적으로는 Vulkan과 DirectX12를 백엔드로 하며, 이를 추상화 하여서 공통된 기능을 제공하는 것을 목표로 함.

> **현재 구현 능력의 한계로 인해서 1차적으로 Vulkan 백엔드만 우선 구현하고, 이후 RDG가 완성되면 DirectX12로 확장할 계획임.**

### Resource Manager
렌더링 할 때 참조의 대상이 되어야 할 여러 자원(buffer, image)를 생성, 소멸, 관리하는 매니저 클래스.

Vulkan Memory Allocator를 추상화 하여, 자원 생성 관련한 메모리 관리를 일임한다.

Dx12의 경우에는 VMA에 상응하는 라이브러리인 D3D12MA를 활용한다.
#### 생성 - factory
모든 종류의 resource는 생성 시점에 resource manager에 자원을 등록해야 함.

따라서 새로운 종류의 resource 생성은 factory에 그에 맞는 생성 method를 추가함을 의미한다.
#### 관리 - bindless
graphrunner 엔진의 자원은 mesh데이터를 제외하면 모두 bindless로 관리한다.

따라서 resource manager는 거대한 descriptor heap과 그에 해당하는 index를 allocation해줘야 한다. 이 index 관리는 bindless한 자원의 생성 시 free한 index를 부여한다.
#### 관리 - residence
현재는 추가적인 고민이 필요하지만, 메모리 자원 상태를 추적하여 메모리가 부족할 경우 상주성 관리를 수행한다. 

> **상주성 관리 쳬계에 대한 추가적인 연구가 필요하다.**

> **상주성 관리의 구현 우선순위는 상대적으로 낮다.** 
#### 소멸 - delete queue
생성 시점에 등록한 정보를 바탕으로 객체 소멸 루틴을 대리수행한다.

이는 gpu와 cpu 사이의 동기화 문제를 고려한 것으로, in-flight frame 수에 기반하여 해당 자원의 소멸을 동기화 한다.

해당 자원의 소멸자가 호출되면, 해당 자원의 핸들을 delete queue에 넣고, 자원의 소멸 안정성이 cpu-gpu 동기화(fence, timeline semaphore)에 의해서 보장되는 순간, 큐의 데이터를 소비한다.

이러한 자원의 소멸자 호출을 위해서 별도의 thread를 두고, 이 thread가 event-driven하게 wake-up하여 자원 소멸을 수행한다.
## Render Graph
렌더링 관련 자원들을 그래프의 형태로 구성 및 관리하는 방법.

어떤 자원이 어느 PSO에 결합되고, 어디서 동기화 되어야 하며, 자원이 언제 유효성이 종료되는지 등을 명확하게 정의할 수 있음.

내 추측에는 다음과 같이 동작함.

root는 비어있음. 이후 depth가 낮을수록 우선순위가 높은 동작이 위치함. 

root 아래로는 render pass가 위치함.

render pass 아래로는 pso가 위치함. pso 교체 비용이 높기 때문에 pso의 binding은 가능한 최상단에 위치.

이후 우선순위가 내려갈수록 그 다음 depth, 그 다음 depth에 들어감.

그 결과로 같은 속성을 가지는 오브젝트일수록 최대한 같은 path을 갖게 됨.

> 렌더링 할 각 오브젝트를 굳이 그래프로 만들 필요가 있을까? 그냥 정렬하면 되는게 아닐까? 렌더 패스는 그래프로 관리하더라도, 렌더 패스에 들어갈 각 오브젝트는 그냥 단순 저장하다 각 오브젝트의 필드의 유사성을 바탕으로 정렬해버리면 되는게 아닌지....

사실 이 부분은 어떻게 보면 trie라고 생각할 수 있음. 다만, 각 field가 character가 되는 거임.

이후 이렇게 만들어진 dag에 대하여 DFS를 수행함. 

DFS과정에서 새로운 node를 밟으면 그에 맞는 행동을 수행하면 됨.

예를 들자면 pso의 교체. 새로이 도달한 node가 pso node이면, 이후에 있을 자식 node들은 동일한 pso를 갖는 것이므로, 이를 위해서 pso를 바인딩 함.

와. 천재적인 발상임.

자원을 graph의 형태로 binding하고, 해당 tree를 순회하면서 command buffer를 초기화 하여 rendering을 수행하다니...

다만, 이 렌더 그래프를 초기화 하려면 해당 객체에 대해서 reflection이 가능해야 할 것으로 보임.
이 부분이 문제가 되겠음. 어떻게 query하지???

===================================

추가적인 연구 결과, render graph의 본질은 render pass 단위의 자원 의존성 추적이었음

앞서 내가 생각한 부분은 dag로 만들었을 때의 이점이긴 하지만, 기존의 render pass가

생각했던 이득은 아닌 것으로 보임. render pass 사이의 의존성을 관리하는 것이 주 목적이었음.

그렇다고 위 내용이 쓰레기는 아닌 것이, 분명 관련된 구현을 상용 영역에서 사용하고 있을 것.

===========================

------------------------------------------

셰이더 -> 셰이더 리플렉션, 어태치먼트 명시 -> 렌더링 오브젝트 팩토리에서 셰이더 리플렉션을 바탕으로 객체 초기화 -> 생성된 객체로 렌더 그래프를 셋업 -> 렌더 그래프 순회, 명령 기록

scene을 정의하는 파일이 있어서, 해당 파일을 읽어 오브젝트 초기화
## Render Queue
렌더링 파이프라인의 교체는 필연적으로 overhead를 가져옴, 

따라서 이 overhead를 해소하기 위해서 파이프라인을 공유하는 객체를 모아서 render call을 수행할 필요가 있음. 

그래서 실제 command buffer에 자원을 제출하기 전에 render queue를 두어서 정렬을 먼저 하는게 좋지 않을까 생각함.

정렬하는 데에 드는 overhead가 얼마나 들까 그 부분이 조금 고민되긴 함.

## Object based Culling
CPU로 렌더링 할 물체에 대해서 절두체 교차 판정을 수행하고, 수행 결과 렌더링 대상이 된다면 그 때 렌더링 수행.

각 물체(정점 군)에 대하여 절두체 교차 판정을 빠르게 수행하는 알고리즘 필요함

혹은 충돌 트리가 있어야 하나???

**Bounding Volume Hierarchy (BVH) / Octree / KD-Tree**  
→ 공간 분할을 해서, 프러스텀 컬링을 트리 탐색처럼 수행.  
→ 수만 개 오브젝트를 하나하나 검사하는 게 아니라, 그룹 단위로 배제 가능.

→ AABB vs Plane 테스트를 SSE/AVX로 벡터화.  
→ 예: 4개의 오브젝트 AABB를 한 번에 테스트.

절두체 컬링은 멀티스레딩도 가능하다고 함. 스레드풀 필요.
## GPU based Occulusion Culling
compute shader를 돌려서 각 물체들의 심도를 비교, 겹침 여부를 판단하여 렌더링 대상에서 제외하는 구현