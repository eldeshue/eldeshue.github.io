---
Date: 2025-08-01
tags:
  - DirectX12
---
# Overview
Dx12의 gpu를 추상화 한 device에 대해서 정리한다.
# Description
가상 어댑터를 나타냅니다. 명령 할당자, 명령 목록, 명령 큐, 펜스, 리소스, 파이프라인 상태 개체, 힙, 루트 서명, 샘플러 및 여러 리소스 뷰를 만드는 데 사용된다.
## DXGIAdapter vs D3D12Device
DXGIFactory에서 생성한 DXGIAdapter와 D3D12Device 둘 다 본질적으로 GPU를 추상화 한 객체이다. 그렇기 때문에 Device가 Adapter로 부터 생성된다. 둘의 차이가 있다면 어떤 layer에서 바라보는 지에 대한 차이가 있을 뿐이다.

DXGIAdapter는 그래픽 관련 low-level layer인 DXGI 관점에서 바라본 GPU이다. 따라서 GPU의 각종 성능 및 수치에 대한 정보를 query할 수 있다. 

반면 D3D12Device는 DX12 컨텍스트 상에서 이해되어야 하는 부분으로, D3D12라는 api에서 요구하는 특정 동작을 추상화 한 부분이다. 그렇기에 각종 api의 상호작용의 핵심이 되는 여러 객체의 생성을 담당하게 된다.

# Reference
- https://learn.microsoft.com/ko-kr/windows/win32/api/d3d12/nn-d3d12-id3d12device
- https://learn.microsoft.com/ko-kr/windows/win32/direct3d12/direct3d-12-interfaces

