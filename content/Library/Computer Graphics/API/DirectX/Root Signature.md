---
Date: 2025-07-31
---
# Overview
pipeline이 수행되는 과정에서 참조할 모든 데이터의 구성을 정의하는 root signature를 정리한다.

# Description
Root Signature, RS는 파이프라인이 실행되는 과정에서 잠조해야 할 모든 종류의 데이터의 배치를 정의하는 객체이다.

중요한 지점은 실제 데이터가 아닌, 배치 방식, 즉 함수에 비교하면 인자에 해당하는 부분이라는 것이다.

> **Root Signature는 lambda의 capture list에 해당한다.**

> **주의 : lambda의 인자에 해당하는 부분은 input layout으로, PSO에 별도로 설정된다.**

RS는 PSO의 일부분으로, GPU가 읽어야 하는 데이터임. 따라서 PSO와 마찬가지로 컴파일 된 바이너리 데이터임.
# API
## Root Parameter
RS를 이루는 요소들. 인덱스로 location을 구분하며, 각 location에 전달되어야 할 view의 타입을 명시해야 함.
``` C++

// Root parameter can be a table, root descriptor or root constants.
// 디스크립터 테이블이 아닌 둘은 간접 접근이 아닌, 직접 접근을 하겠다는 의미임.
CD3DX12_ROOT_PARAMETER slotRootParameter[1];
```
### Descriptor Table
``` C++

// Create a single descriptor table of CBVs.
CD3DX12_DESCRIPTOR_RANGE cbvTable;
cbvTable.Init(D3D12_DESCRIPTOR_RANGE_TYPE_CBV, 1, 0);
slotRootParameter[0].InitAsDescriptorTable(1, &cbvTable);
```
GPU로 하여금 descriptor table을 거쳐서 참조하였으면 하는 데이터를 정의하는 방법.

desc table을 생성하고, 여기에 desc의 타입을 명시하여 슬롯을 초기화 함.
### Root Descriptor

### Root Constant

## Example, Creating Root Signature
RS를 생성하기 위해서 초기화 하는 구조체.

``` C++
// RS 생성을 위한 desc 초기화
// A root signature is an array of root parameters.
CD3DX12_ROOT_SIGNATURE_DESC rootSigDesc(1, slotRootParameter, 0, nullptr,
	D3D12_ROOT_SIGNATURE_FLAG_ALLOW_INPUT_ASSEMBLER_INPUT_LAYOUT);

// RS를 생성하기 위한 메모리 할당
// RS는 GPU를 위해서 컴파일 된 바이너리 데이터임
// create a root signature with a single slot which points to a descriptor range consisting of a single constant buffer
ComPtr<ID3DBlob> serializedRootSig = nullptr;
ComPtr<ID3DBlob> errorBlob = nullptr;
HRESULT hr = D3D12SerializeRootSignature(&rootSigDesc, D3D_ROOT_SIGNATURE_VERSION_1,
	serializedRootSig.GetAddressOf(), errorBlob.GetAddressOf());

if (errorBlob != nullptr)
{
	::OutputDebugStringA((char*)errorBlob->GetBufferPointer());
}
ThrowIfFailed(hr);

// RS생성
ThrowIfFailed(md3dDevice->CreateRootSignature(
	0,
	serializedRootSig->GetBufferPointer(),
	serializedRootSig->GetBufferSize(),
	IID_PPV_ARGS(&mRootSignature)));

```


