---
Date:
---
# Overview
GPU가 수행할 명령을 기록하는 buffer, command list에 대해서 정리한다.
# Description
cmdlist는 Dx12의 핵심 기능 중 하나로,  gpu가 수행해야 할 명령 절차를 **녹화**하는 대상이다.  CPU-GPU의 비동기 프로그래밍의 핵심적인 구조를 담당하고 있다.

과거 api의 호출이 gpu의 동작을 즉각 요청하는 구조였다면, DX12에서의 api호출은 대부분 cmd list에 대한 명령 기록으로 구성된다. cpu는 gpu가 수행할 동작을 cmd list에 작성하고, 이후 이 cmdlist가 cmd queue에 제출되면 그제서야 gpu가 순차적으로 명령을 수행하게 된다. 

  gpu의 큐에 제출할 명령을 기록만 할 뿐, 기록 시점에서는 동작이 전혀 발생하지 않는다. 뿐만 아니라, 큐에 제출하는 시점에서도 동작 여부는 보장할 수 없다. gpu의 큐에 명령이 얼마나 쌓여있는지 알 수 없기 때문이다.

이러한 cmd list는 cmd allocator에서 할당된 메모리 공간에 위치하게 된다.
# API
대부분의 렌더링 관련 동작이 cmd list를 중심으로 이루어진다. 따라서, 그 api의 양도 방대하여 정리하기가 마땅치 않다. 다음의 문서를 참고할것.

- MSDN : https://learn.microsoft.com/ko-kr/windows/win32/api/d3d12/nn-d3d12-id3d12graphicscommandlist