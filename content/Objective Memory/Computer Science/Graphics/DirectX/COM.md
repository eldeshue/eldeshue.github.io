---
Date: 2025-07-24
tags:
  - DirectX
---
# COM이란 무엇인가?
`COM`(**Component Object Model**)은 DirectX를 비롯한 Windows API의 근간을 이루는 핵심 기술 중 하나입니다. COM은 단순한 기술이 아니라 **Windows 전반의 객체 관리, 재사용성, 바이너리 수준 호환성**을 위한 **런타임 시스템**
## Component
**COM에서 "컴포넌트"란?**

> **이진 형식(예: DLL)에 배포 가능한 독립적 객체 단위**  
>  **바이너리 수준에서 재사용 가능한 프로그램 단위**입니다.

C++ 등의 언어로 작성된 클래스가 소프트웨어 수준의 객체라면, component는 실행 가능한 바이너리의 형태로 존재하는 객체임. 즉, 단순 소프트웨어가 아닌 여러 장치의 드라이버나 OS의 특정 기능 등을 의미함. 이 때 컴포넌트에 응용프로그램이 접근하기 위한 api가 COM임.
## COM 인터페이스
### Implementation - Binary
**실제 COM이 가리키는 객체는 DLL 혹은 EXE의 형태로 존재**하며, **OS에 등록(레지스트리)되어 관리**되고, 다른 **응용프로그램이 loading하여 접근**할 수 있음. 이를 인터페이스의 형태로 객체 지향적으로 관리가 가능함.
### IUnknown 인터페이스
모든 COM이 상속하는 최상단 인터페이스. 
``` C++
// unknwn.h
class IUnknown {
public:
    virtual ULONG AddRef() = 0;
    virtual ULONG Release() = 0;
    virtual HRESULT QueryInterface(REFIID riid, void** ppv) = 0;
};
```
#### 1) AddRef
자원에 대한 참조 count 증가.
#### 2) Release
자원에 대한 참조 count 감소. 0이 되면 자원이 해제됨.
#### 3) QueryInterface
- `riid` : 쿼리를 수행할 interface의 id. **input**
- ppv : `riid`가 가리키는 인터페이스의 포인터의 주소. **output**
COM객체에 대하여 쿼리를 수행,  `riid`값을 바탕으로 vtable을 탐색하여 실제 com 구현체의 pointer를 ppv가 가리키는 포인터에 할당하는 메서드임. 

> **따라서, 적절한 IID를 줬다면, 호출한 자기 자신의 포인터와 동일한 값을 ppv에 넣게 됨.**

COM은 인터페이스 기반이긴 하지만, 언어에 독립적임. 따라서 C++의 RTTI를 쓸 수 없음. 그래서 실제 인터페이스를 식별하는 값인 IID를 전달하여 명시적으로 down casting을 수행함.
##### IID_PPV_ARGS
앞서 말한 IID를 명시적으로 전달하지 말고, ppv가 가리키는 포인터가 가리키는 값을 바탕으로 IID를 제공, 확장하는 매크로.
``` C++
ID3D12Device* device = nullptr;

// use macro
D3D12CreateDevice(..., IID_PPV_ARGS(&device));

// after expansion
// both are same
D3D12CreateDevice(..., __uuidof(ID3D12Device), (void**)&device);

// Mistakes without macro
// dangerous, allocate ID3D12Resource to ID3D12Device, not match
device->CreateCommittedResource(..., __uuidof(ID3D12Resource), (void**)&device);

```

riid와 ppv의 실제 타입이 불일치하는 문제를 막기 위함임.
# Example - ID3D12Device
COM의 대표적인 예시가 DX12에서 필수적으로 사용하는 `ID3D12Device`다. DX12는 그래픽 api이고, 이는 gpu 드라이버에서 구현되어 있다. 따라서 그래픽 api를 사용하기 위해서는, gpu의 드라이버에 접근할 필요가 있는데, windows는 이를 com으로 제공한다.
``` C++
// ID3D12Device com 객체를 불러서 device라는 com_ptr에 할당
D3D12CreateDevice(..., IID_PPV_ARGS(&device));
```
위 코드는 내부적으로 QueryInterface를 호출할 것이며, 이를 통해서 ID3D12Device라는 COM객체를 load하는데, 이 COM객체의 인터페이스를 통해서 바이너리 형태의 gpu driver를 호출합니다.

> 즉, GPU제조사가 바이너리(드라이버)의 형태로 제공하는 api를 응용프로그램이 사용하기 위한 인터페이스.

