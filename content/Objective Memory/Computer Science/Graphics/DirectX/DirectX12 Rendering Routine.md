---
Date: 2025-08-03
tags:
  - DirectX12
  - Graphics
---
# Overview
Dx12 api를 활용한 렌더링 루틴의 전반적인 거동에 대해서 정리한다.

Dx12를 바탕으로 렌더링 루틴을 작성하기 위해서 어떤 과정을 수행해야 하는지 대략적으로 다룬다.

각 과정의 상세에 대해서는 연결된 link를 통해서 탐색.
# Description
## 1 Dx12 Context Initialization

### 1.1 DXGI
어댑터 선정, 기능 테스트, swap chain, 등
### 1.2 D3D12
디바이스, 커맨드 얼로케이터, 커맨드 리스트, 커맨드 큐 등
## 2 Resource Initialization
렌더링 과정에서 파이프라인의 입력 및 참조에 사용될 자원의 초기화를 수행한다. 
### 2.1 Heap
vertex buffer, index buffer, const buffer, texture buffer, render target buffer(swap chain), depth-stencil buffer, ...
### 2.2 Descriptor heap & Descriptor(View)
생성한 여러 버퍼의 desc를 저장할 힙과, 그 힙에 desc 저장
### 2.3 Pipeline
렌더링 파이프라인의 정적인 요소를 기술하는 pso를 만드는 과정.

shader를 중심으로, 이 shader에 어떤 자원을 어떻게 공급하는가 에 대한 정의임.
#### 2.3.1 Input Layout
pipeline에 공급할 입력을 정의함.

vertex buffer와 index buffer의 layout, geometry topology등이 정의됨.
#### 2.3.2 Root Signature
shader가 참조할 자원이 어떻게 배치되어야 하는지 기술함. 
#### 2.3.3 PSO
앞서 지정한 요소를 바탕으로 렌더링 파이프라인 자체를 정의하는 pso를 생성.

복수의 pso를 운영해야 한다면, 이를 pipeline library에 저장해야 함.

셰이더를 컴파일 하여 공급하거나 컴파일되어 캐싱된 셰이더를 deserialize해야 함.
## 3 Rendering Routine
앞서 준비한 자원과 pso를 바탕으로 렌더링을 명령을 녹화, 제출.
### 3.1 Recording
pso 선정, 여러 자원 바인딩, 자원 상태 전환, 등

여기서 스레드 풀 도입하여 복수의 커맨드 리스트를 독립적으로 초기화.
커맨드 리스트 사이에서는 의존성이 없어야 할 것으로 보임.

만약 render queue가 구현되었다면, 여기서 기록하기 이전에 정렬 필요.
### 3.2 Submit CmdList to CmdQueue
큐에 초기화 한 리스트들 제출
# Summary