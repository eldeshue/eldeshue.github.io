---
Date: 2025-07-14
tags:
  - DirectX
---
# Overview
윈도우즈에서 그래픽 출력을 위해 추상화된 계층, DXGI에 대해서 다룬다.
# DirectX Graphic Infrastructure
DXGI는 Direct3D와 함께 쓰이는 API, 여러 그래픽 API들에 공통인 그래픽 관련 작업들이 존재하는 것들을 한데 묶은 것. 교환 사슬(_swap chain_)이나 페이지 전환, 어탭터, 모니터, 디스플레이 모드 등의 공통된 그래픽 시스템에 대한 api를 제공한다.

> MSDN : DXGI (Microsoft DirectX Graphics Infrastructure)는 그래픽 어댑터 열거, 디스플레이 모드 열거, 버퍼 형식 선택, 프로세스 간(예 : 응용 프로그램과 데스크탑 창 관리자(DWM) 간) 자원 공유, 렌더링된 프레임을 디스플레이하기 위한 창 또는 모니터에 표시하는 작업을 처리합니다.

DXGI는 이러한 low-level 작업을 처리하는 layer로, **커널 모드 드라이버 및 시스템 하드웨어와 통신한다.**
# Contents
DXGI는 다음의 요소들을 관리함.
## Adapter
**어댑터는 컴퓨터의 하드웨어 및 소프트웨어 기능을 추상화한 것입니다.** 일부 장치는 비디오 카드와 같은 하드웨어로 구현되고 일부 장치는 Direct3D 참조 래스터라이저와 같은 소프트웨어로 구현됩니다. 어댑터는 그래픽 응용 프로그램에서 사용하는 기능을 구현합니다.

DXGI는 이러한 어댑터를 열거하여 정보를 획득하게 할 수 있다. 

## Presentation - swap chain
DXGI의 또 다른 핵심적인 기능은 렌더링한 프레임 버퍼를 출력하는 것이다. 즉, DXGI는 2D 이미지 데이터를 디스플레이 드라이버에게 전달하는 부분을 대리하며,이를 presentation이라 한다.

과거에는 프레임 버퍼가 한 개로, 렌더링 과정이 그대로 화면에 출력되었다. 그러나 현대에는 프레임 버퍼가 두 개 이상이며, 하나의 버퍼가 출력(presentation)되는 동안, 다른 버퍼(흔히 후면 버퍼라 부르는)에 렌더링을 수행한다. 이후 렌더링이 완료되면 presentation되는 것과 교체(swap)되어 앞서 동작을 반복한다.

> **출력의 대상이 되는 것을 front buffer, 렌더링의 대상이 되는 쪽을 back buffer라 부른다.**

이러한 렌더링-출력의 연쇄적인 동작을 수행하는 일련의 버퍼들을 **swap chain**이라 부르며, DXGI가 이를 제공한다.
# API
## IDXGIFactory
**그래픽 디바이스, 어댑터(GPU), 스왑 체인(swap chain)** 등을 생성하거나 열거(enumerate)하는 데 사용되는 객체. 최초 초기화 과정에서 dxgi 팩토리를 먼저 만들고, 이후 다른 초기화를 수행한다.

> **DXGI layer를 추상화한 객체가 바로 IDXGIFactory이다.**

DXGI는 OS나 DirectX의 발전에 따라서 함께 발전하여 다음과 같은 여러 layer가 존재함.
### Version 별 비교

| 버전   | 인터페이스 이름        | 등장 버전             | 주요 기능 / 변화                                                          |
| ---- | --------------- | ----------------- | ------------------------------------------------------------------- |
| v1   | `IDXGIFactory`  | DXGI 1.0 (D3D10)  | 기본 팩토리. 어댑터/스왑체인 열거 및 생성                                            |
| v1.1 | `IDXGIFactory1` | DXGI 1.1 (D3D11)  | 어댑터 명칭, `IsCurrent()` 등 기능 추가                                       |
| v2   | `IDXGIFactory2` | DXGI 1.2 (Win8)   | `CreateSwapChainForHwnd`, `CreateSwapChainForCoreWindow` 등 WinRT 연동 |
| v3   | `IDXGIFactory3` | DXGI 1.3 (Win8.1) | 거의 변화 없음                                                            |
| v4   | `IDXGIFactory4` | DXGI 1.4 (Win10)  | `EnumAdapters1` → `EnumAdaptersByLuid`, `CheckFeatureSupport` 등 추가  |
| v5   | `IDXGIFactory5` | DXGI 1.5 (Win10)  | `RegisterAdaptersChangedEvent` 등 알림 기능                              |
| v6   | `IDXGIFactory6` | DXGI 1.6 (Win10)  | GPU 성능 기반 우선순위 열거 (`EnumAdapterByGpuPreference`)                    |
| v7   | `IDXGIFactory7` | DXGI 1.7 (Win11)  | 현재 거의 사용 안 됨, 문서화도 적음                                               |
#### 주요 버전 상세
##### 🧩 `IDXGIFactory4` (DXGI 1.4, Win10)
- `EnumAdaptersByLuid()` 도입 — 특정 LUID 기반 GPU 선택 가능
- `CheckFeatureSupport()` 지원 → 어댑터/시스템의 기능을 미리 확인할 수 있음
- D3D12와 연동 시 많이 쓰이는 버전
##### 🧩 `IDXGIFactory5` (DXGI 1.5)
- `RegisterAdaptersChangedEvent()` 추가 → 외장 GPU가 연결/해제되었을 때 알림받을 수 있음
    
- 핫플러그 이벤트 대응
##### 🧩 `IDXGIFactory6` (DXGI 1.6, Win10 RS4)
- **GPU 우선순위 기반 열거** (`EnumAdapterByGpuPreference`)
    - 예: `DXGI_GPU_PREFERENCE_HIGH_PERFORMANCE`, `MINIMUM_POWER`, 등
- 노트북에서 내장 vs 외장 GPU 선택을 코드로 가능

   ---

## IDXGIAdapter
그래픽 장치(GPU 혹은 가상 GPU)를 추상화 한 COM. 팩토리로 어댑터를 검색한 다음, 이 어댑터로 D3D12Device를 생성함.  어댑터를 통해서 해당 장치의 성능 및 지원하는 기능 등을 확인할 수 있음.

``` C++
ComPtr<IDXGIAdapter1> adapter;
factory->EnumAdapters1(0, &adapter); // 디바이스 어댑터 획득

D3D12CreateDevice(adapter.Get(), ...); // 해당 디바이스로 D3D12 컨텍스트 생성
```

마찬가지로 version에 따라서 여러 인터페이스가 존재함.

| 버전       | 인터페이스 이름        | 특징                           |
| -------- | --------------- | ---------------------------- |
| DXGI 1.0 | `IDXGIAdapter`  | 가장 기본. DXGI 1.0용             |
| DXGI 1.1 | `IDXGIAdapter1` | 상세 정보(예: 전용 VRAM 크기 등) 조회 가능 |
| DXGI 1.4 | `IDXGIAdapter2` | Windows 10에서 드라이버 노출 제어용     |
| DXGI 1.6 | `IDXGIAdapter3` | GPU 메모리 사용량 추적 기능 등 추가       |

---

| IDXGIAdapter1 기준 주요 메서드                    | 설명                                    |
| ------------------------------------------ | ------------------------------------- |
| `GetDesc1()`                               | 어댑터의 상세 정보(DXGI_ADAPTER_DESC1 구조체) 반환 |
| `EnumOutputs()`                            | 해당 어댑터에 연결된 출력 장치(모니터 등)를 열거          |
| `CheckInterfaceSupport()`                  | D3D 지원 여부 확인                          |
| `QueryVideoMemoryInfo()` (`IDXGIAdapter3`) | VRAM 사용량 정보 획득 (DX12 최적화 시 중요)        |
- **DX12 디바이스는 반드시 IDXGIAdapter1 이상을 요구**함.  
    보통 `IDXGIFactory6::EnumAdapterByGpuPreference()`를 써서 `IDXGIAdapter1`을 반환받고, 이걸로 D3D 디바이스 생성.
- **WARP 디바이스**가 필요한 경우엔 `DXGI_ADAPTER_FLAG_SOFTWARE`가 설정된 어댑터를 찾아야 함.
- CreateDevice에서 어댑터로 nullptr을 주면 0번 어댑터로 디바이스 컨텍스트를 생성함.