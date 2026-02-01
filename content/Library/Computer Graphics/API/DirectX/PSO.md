---
Date: 2025-07-31
tags:
  - DirectX12
---
# Overview
렌더링 파이프라인 그 자체를 추상화 한 객체, Pipeline State Object에 대해서 정리한다.
# Description
셰이더를 비롯하여 IA, RS, 샘플러 등, 렌더링을 위해서 구체적으로 설정해야 하는 모든 정젹(static) 요소를 정의해둔 객체. 동적인 요소와는 분리됨.

> **pipeline이 프로세스라면, PSO는 Process Control Block에 해당함.**

따라서 PSO의 교체는 context switching에 준하는 행동으로, 비용(cache miss 등)이 발생함.

렌더링 과정에서 물체의 material에 따른 shader 변경에 의해서 복수의 pipeline을 관리해야 하는 문제가 생길 수 있는데, 이를 위해서 D3D12PipelineLibrary를 제공하여 관리를 수월하게 할 수 있다.
# API