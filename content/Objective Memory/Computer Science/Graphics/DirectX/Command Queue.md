---
Date: 2025-07-23
tags:
  - DirectX12
  - Graphics
  - DirectX
---
# Overview
GPU가 수행할 명령의 대기열, Command Queue에 대하여 정리한다.
# API
## 생성
``` C++

D3D12_COMMAND_QUEUE_DESC queueDesc = {};
queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;	// 다이렉트 리스트
queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
ThrowIfFailed(md3dDevice->CreateCommandQueue(&queueDesc, IID_PPV_ARGS(&mCommandQueue)));
```
### D3D12_COMMAND_QUEUE_DESC
커맨드 큐 생성을 위한 디스크립터.

## CommandList 등록

