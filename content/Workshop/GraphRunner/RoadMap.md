---
tags:
  - Graphics
  - Vulkan
  - DirectX12
---
# Description
모던 그래픽스 api로 구현하는 그래픽 엔진(게임 엔진).

지금까지 학습한 모든 요소를 총집한 나만의 필살기, 키스톤 프로젝트.
# Goal
앞으로의 학습에서 구현하게 될 그래픽적 요소의 기반이 될 엔진임.

나의 기술적 성장에 발맞춰 함께 성장할 프로젝트임.

현재는 다음의 요소를 구현 목표로 고려하고 있음.
- 크로스플랫폼 RHI(Vulkan + DirectX12)
- 렌더 그래프 기반의 렌더링 컴포넌트 관리
- 옥트리 기반 오브젝트 관리
- 디퍼드 렌더링

# Sample Road Map
## Phase 1 : 개발 환경 및 Utility 구축

- CMake 기반의 크로스 플랫폼 빌드 환경 구성(완료)
- 유틸리티 
	- Logging System
	- assertion(?)
## Phase 2 : RHI 구현 및 검증
모던 그래픽 api를 구성하는 요소를 추상화, Vulkan과 Direct12를 통합한다.

Vulkan을 중심으로 구현을 진행한 다음, 이후에 Direct12를 통합한다.

**추상화 대상:**
- `Instance`, `Device`, `Queue`
- `CommandBuffer`, `CommandPool`
- `Buffer` (Vertex, Index, Uniform Buffer 등)
- `Texture` (2D, Cube Map 등) 및 `Sampler`
- `Shader` 및 `Pipeline State Object (PSO)`
- `Descriptor Set` (Vulkan) / `Root Signature` (DX12)의 추상화 (가장 까다로운 부분 중 하나)
- 동기화 프리미티브: `Fence`, `Semaphore`

RHI 구현이 끝나는 대로, RHI를 활용한 간단한 렌더링 루틴을 작성, 실제 렌더링 여부를 확인한다.
- 삼각형 그리기
- wavefront-obj 렌더링
## Phase 3 : RDG 구현 및 검증
모던 그래픽 엔진의 핵심 아키텍쳐인 RDG를 구현한다.

RDG는 렌더 패스를 기준으로 자원을 관리한다.

여기서 내 나름의 추가적인 고민은 객체에 존재하는 렌더링 관련 컴포넌트를 쿼리, 
각 컴포넌트를 노드로 삼는, 고도화된 DAG를 구성하도록 하는 계획임.

RDG를 바탕으로 렌더링 자원 관리를 수행한다.

> **RDG 관련 학습 및 연구 필요함**

이 시점까지 완료가 됐으면 기반이 되는 구성은 모두 완료된 부분이라고 생각됨.

이후에는 이 기반 위에 여러 기능을 추가하는 방향으로 진행 가능하다고 생각됨
## Phase 4 : Advanced 구현
phase 3까지 구현한 기반 위로 엔진으로서 가져야 할 여러 고급 기능을 추가하는 부분.

현재 고려하는 고급 기능은
- GUI 통합 : gui는 다른 기능 개발에 큰 도움을 줌. 따라서 gui를 빠르게 통합하는 것이 상당한 도움이 됨.
- Defered Rendering
- Octree로 공간 분할, 절두체 컬링
- 셰이더 관리 시스템
	- Shader Reflection에 기반한 동적 오브젝트 로딩
	- shader를 읽어서 객체 파이프라인을 설정하는 시스템
	- shader 크로스 
- 게임 엔진처럼 스크립팅(?)