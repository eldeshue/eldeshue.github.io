---
Date: 2025-08-01
tags:
  - DirectX12
---
# Overview
Dx12에서 정의하는 각종 자원의 공통 인터페이스인 ID3DResourc에 대해서 정리한다.
# Description
MSDN에 따르면 다음과 같다.
> 리소스는 GPU 실제 메모리의 사용을 추상화하는 Direct3D 개념입니다. 리소스에는 실제 메모리에 액세스하기 위한 GPU 가상 주소 공간이 필요합니다. 리소스 생성은 자유 스레드 방식으로 진행됩니다.

또는
> CPU 및 GPU의 일반화된 기능을 캡슐화하여 실제 메모리 또는 힙을 읽고 씁니다. 셰이더 샘플링에 최적화된 다차원 데이터뿐만 아니라 간단한 데이터 배열을 구성하고 조작하기 위한 추상화가 포함되어 있습니다.

Dx12에서 말하는 자원은 CPU와 GPU가 공유해야 하는 종류의 데이터를 말한다. 즉 vertex, index, texture, back buffer, front buffer, constant buffer, depth stencil buffer, ... 등 모든 것이다. 

Dx12의 자원은 CPU가 공급하게 되지만, 최종적으로는 GPU에 의해 참조되어야 한다. GPU에 의한 참조는 descriptor에 의해서 수행되기는 하지만, 어쨋건 이들 자원은 CPU와 GPU양쪽에 속한 데이터가 된다. D3DResource 인터페이스가 존재하는 이유는 바로 이 cpu-gpu 양쪽에 속하는 바로 이 특성 때문이다. 

따라서 D3D12Resource는 단순 메모리 공간에 더해서 이러한 복잡한 문제를 해결하는 데에 도움이 되는 여러 api를 제공한다. 
## Alignment
heap에 데이터를 저장할 때, alignment를 맞춰줘야 하는 경우가 왕왕있다. 이 부분 주의해야 할 것으로 보임.
- conatant buffer의 데이터는 256byte 
- sub resource는 512byte
- index buffer의 경우는 해당 인덱스 포맷의 배수
- etc
## Resource의 세 가지 유형
Resource는 사용을 위해서 GPU virtual address가 필요한데, 이와 관련하여 다음의 세 종류 유형이 있다.
### Committed Resource
> 커밋된 리소스는 세대에 걸쳐 Direct3D 리소스의 가장 일반적인 아이디어입니다. 이러한 리소스를 만들면 전체 리소스에 맞는 충분히 큰 암시적 힙에 해당하는 가상 주소 범위가 할당되고, 가상 주소 범위가 힙이 캡슐화하는 실제 메모리로 커밋됩니다. 이전 Direct3D 버전과 기능 패리티를 일치하려면 암시적 힙 속성을 전달해야 합니다.

대부분의 자원을 committed resource로 생성하는 만큼, 가장 기본적인 자원 할당 방법. 사용할 크기의 메모리에 해당하는 가상/실물 메모리를 할당 받음. 
### Reserved Resource
> 예약된 리소스는 Direct3D 11 타일형 리소스와 동일합니다. 만들 때 가상 주소 범위만 할당되고 힙에 매핑되지 않습니다. 애플리케이션은 나중에 이러한 리소스를 힙에 매핑합니다. 이러한 리소스의 기능은 UpdateTileMappings**를 사용하여 64KB 타일 세분성의 힙에 매핑될 수 있으므로 현재 Direct3D 11에서**변경되지 않습니다.

[[Tile and Tile Resource | Tile]]이란 페이징을 통해서 demand loading하는 테크닉으로, 해당 주소 영역을 미래에 사용할 것이라고 범위에 대한 가상 메모리만 할당만 받고, 매핑, 즉 실물 메모리는 할당받지 않음을 의미합니다.
### Placed Resource
> Direct3D 12의 새로운 기능으로 리소스와 별도로 힙을 만들 수 있습니다. 그런 다음, 단일 힙 내에서 여러 리소스를 찾을 수 있습니다. 타일형 또는 예약된 리소스를 만들지 않고도 이 작업을 수행할 수 있으므로 애플리케이션에서 직접 만들 수 있는 모든 리소스 종류에 대한 기능을 사용할 수 있습니다. 여러 리소스가 겹칠 수 있으며 실제 메모리를 올바르게 다시 사용하려면 ID3D12GraphicsCommandList::ResourceBarrier[**를 사용해야**](https://learn.microsoft.com/ko-kr/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-resourcebarrier) 합니다.

리소스와 분리된 힙이란 하나의 힙에 여러 ID3D12Resource를 달아줄 수 있음을 의미함. 즉 resource-barrier등을 통한 상태 변경을 통해서 하나의 실체에 대한 여러 resource 인터페이스를 두어서 재사용할 수 있음을 의미하는 것으로 보임.

해당 힙에 대한 소유권을 가지게 되는지 궁금함...

> **겹침이라는 표현을 볼 때, 한 메모리 영역을 다양한 방식으로 사용하는 것으로 보임. 추가적인 연구가 필요함**
# API
##  ID3DResource
### Map & UnMap
Map 메서드는 해당 Resource 인터페이스가 가리키는 gpu자원에 대하여 **cpu가 접근할 수 있도록 memory mapping된 pointer**를 반환한다. 첫 호출에서는 gpu 자원을 시스템 메모리로 memory mapping하고, 이후 호출부터는 참조 count를 증가시킨다. 

UnMap메서드는 기본적으로 참조 count를 감소시키며, 참조 count가 0이 되면 resource와 시스템 메모리 사이의 mapping을 해제한다. Map으로 받아낸 포인터는 단순 포인터이므로, 해당 포인터의 scope를 벗어나기 전에 UnMap을 호출하여 참조 count를 줄여줘야 함.

> **Map과 UnMap은 반드시 짝을 맞춰 사용하여 참조 count를 줄여줘야 한다.**

즉, GPU의 실제 메모리(VRAM 또는 AGP/PCIe 버스를 통해 접근 가능한 시스템 RAM 영역)에 있는 리소스의 특정 부분이, CPU가 접근할 수 있는 가상 주소 공간(시스템 RAM 주소 공간)에 연결되는 것이다. 이 과정은 시스템 버스를 통해 GPU 메모리에 접근하기 위한 물리적인 주소 변환과정이 포함될 수 있다.

다만, 해당 pointer를 통한 자원의 수정은 동기화를 보장하지 않는다. 따라서 적절한 동기화를 수행한 다음, 해당 포인터에 대한 접근을 수행하는 것이 옳다. 다만, **참조 카운트가 0이 되어 mapping이 해제되면 모든 변화가 완전히 반영된다.**

#### Wrapper Class
앞서 살펴본 바와 같이, Map과 Unmap은 짝을 이루어 참조 count를 제어해야 마땅하다. 다만, 이들은 단순 pointer이므로 RAII패턴을 통해서 짝을 맞춰주면 보다 에러를 줄일 수 있다.

이러한 에러를 줄이기 위한 unique pointer like한 wrapper를 생각해볼 수 있는데, 이는 다음과 같다.
``` C++
/*
	Gemini가 생성함. 예시 정도로만 생각할 것.
*/
#include <d3d12.h>
#include <cstdint> // for size_t

class D3D12MappedResource
{
public:
    // 생성자: ID3D12Resource::Map을 호출하고 포인터를 저장
    D3D12MappedResource(
        ID3D12Resource* pResource,
        UINT subresource = 0,
        const D3D12_RANGE* pReadRange = nullptr)
        : m_pResource(pResource),
          m_subresource(subresource)
    {
        if (m_pResource)
        {
            // Map 호출. pReadRange는 null일 수 있음 (CPU가 읽을 필요 없는 경우)
            // pReadRange는 Map의 마지막 인자. Unmap 시 플러시할 영역을 좁혀 성능 향상 가능
            HRESULT hr = m_pResource->Map(m_subresource, pReadRange, reinterpret_cast<void**>(&m_pData));
            if (FAILED(hr))
            {
                // 에러 처리: 예외를 던지거나 nullptr로 설정
                m_pData = nullptr;
                // 실제 환경에서는 예외 처리 또는 로깅 필요
            }
        }
    }

    // 소멸자: ID3D12Resource::Unmap을 호출하여 매핑 해제
    ~D3D12MappedResource()
    {
        if (m_pResource && m_pData)
        {
            // Unmap 호출. pWrittenRange는 null일 수 있음 (CPU가 쓰지 않은 경우)
            // CPU가 쓴 영역만 지정하여 플러시 오버헤드 감소 가능
            m_pResource->Unmap(m_subresource, nullptr); // 또는 D3D12_RANGE{0, 0}
        }
    }

    // 복사 생성자 및 할당 연산자 삭제 (리소스 소유권 문제 방지)
	// move only
    D3D12MappedResource(const D3D12MappedResource&) = delete;
    D3D12MappedResource& operator=(const D3D12MappedResource&) = delete;

    // 이동 생성자 및 할당 연산자 (옵션: 소유권 이동을 허용)
    D3D12MappedResource(D3D12MappedResource&& other) noexcept
        : m_pResource(other.m_pResource),
          m_pData(other.m_pData),
          m_subresource(other.m_subresource)
    {
        other.m_pResource = nullptr;
        other.m_pData = nullptr;
    }

    D3D12MappedResource& operator=(D3D12MappedResource&& other) noexcept
    {
        if (this != &other)
        {
            // 기존 리소스 언매핑
            if (m_pResource && m_pData)
            {
                m_pResource->Unmap(m_subresource, nullptr);
            }

            m_pResource = other.m_pResource;
            m_pData = other.m_pData;
            m_subresource = other.m_subresource;

            other.m_pResource = nullptr;
            other.m_pData = nullptr;
        }
        return *this;
    }


    // 매핑된 데이터에 접근하기 위한 포인터 반환
    void* GetData() const { return m_pData; }

    // 타입 캐스팅 연산자 (선택적)
    operator void*() const { return m_pData; }

    template<typename T>
    T* As() const { return static_cast<T*>(m_pData); }

private:
    ID3D12Resource* m_pResource; // 매핑된 리소스 객체 (소유권은 갖지 않음)
    void* m_pData;               // Map 결과 얻은 포인터
    UINT  m_subresource;         // 매핑된 서브 리소스 인덱스
};

// 사용 예시:
void UpdateConstantBuffer(ID3D12Resource* pConstantBuffer)
{
    // 스코프 내에서 D3D12MappedResource 객체 생성
    // 생성과 동시에 Map 호출
    D3D12MappedResource mappedCBuffer(pConstantBuffer);

    if (mappedCBuffer.GetData())
    {
        // 상수 버퍼 구조체 정의
        struct MyConstantBufferData
        {
            // ... 데이터 멤버들 ...
            float someValue;
        };

        // 데이터 복사 또는 수정
        MyConstantBufferData* pCBufferData = mappedCBuffer.As<MyConstantBufferData>();
        pCBufferData->someValue = 1.0f;
        // memcpy(mappedCBuffer.GetData(), &newData, sizeof(newData));
    }
    // 스코프를 벗어나면 mappedCBuffer의 소멸자가 호출되고 Unmap 자동 호출
}

// 또는 수동으로 범위 지정 Unmap
void UpdateConstantBufferWithRange(ID3D12Resource* pConstantBuffer)
{
    struct MyConstantBufferData
    {
        float val1;
        float val2;
        float val3;
        // ...
    };

    D3D12_RANGE readRange = { 0, 0 }; // CPU가 이 리소스를 읽지 않으므로
    D3D12MappedResource mappedCBuffer(pConstantBuffer, 0, &readRange);

    if (mappedCBuffer.GetData())
    {
	    // constant buffer로 casting하여 상호작용
	    // 아마 원본은 upload heap일 것
        MyConstantBufferData* pCBufferData = mappedCBuffer.As<MyConstantBufferData>();
        pCBufferData->val1 = 1.0f;
        pCBufferData->val2 = 2.0f;

        // CPU가 쓴 영역을 명시적으로 지정하여 Unmap 오버헤드 감소
        // Unmap의 두 번째 인자는 D3D12_RANGE* pWrittenRange
        D3D12_RANGE writtenRange = { offsetof(MyConstantBufferData, val1), offsetof(MyConstantBufferData, val2) + sizeof(float) };
        // 그러나 D3D12MappedResource 소멸자에서 이를 직접 처리하려면
        // Unmap 호출 시 해당 정보를 넘겨줄 방법이 필요함.
        // 이 래퍼는 편의상 nullptr로 호출함. 필요하다면 Unmap 메소드를 추가하여 호출.
    }
}
```


# Reference
- heap의 활용 방법 : https://learn.microsoft.com/ko-kr/windows/win32/direct3d12/large-buffers