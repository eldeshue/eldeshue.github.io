---
Date: 2025-08-17
tags:
  - Graphics
---
# Overview
여러 렌더링 엔진 및 렌더러의 기본적인 구조에 대해서 다룬다.

# Real Time Rendering
렌더링이란 결국 2D 버퍼에 픽셀값을 채워 넣는 행위이다. 이렇게 생성된 것을 결국 이미지이며, 이를 display에 출력하게 된다.

그래픽스란, 이러한 이미지를 실시간으로 생성하는 것에 주안점을 둔다. 따라서 **성능**이 중요하다.
# Pipeline
이러한 렌더링은 gpu라는 전용 프로세서에서 수행된다. gpu는 그래픽 관련 연산을 전문으로 처리하는 프로세서로, 대단히 많은 코어를 가져서 높은 수준의 병렬 연산에 특화되어 있다. 

shader는 이 gpu를 구성하는 무수한 코어에서 병렬로 실행되는 코드이며, 이 shader에 데이터를 공급하고, 그 결과를 받는 흐름 자체를 pipeline이라 부른다. 결국 graphics api란 이 pipeline을 운영하는 것이다.
## Category of Pipeline

이 pipeline에는 여러 종류가 있는데, 일단 3가지를 주로 고려한다(더 있는지 잘 모름).
- Graphics Pipeline : 흔히 다루는 레스터라이징 기법을 위한 파이프라인, 3D 렌더링을 위해서 고정된 부분이 존재한다.
- Compute Pipeline : 여러 복합적인 목적을 위해서 사용되는 범용 계산 파이프라인, 프로그래머빌리티가 높음.
- Raytracing Pipeline : 레이트레이싱을 위한 전용 파이프라인.
## Shader
결국 gpu가 실행하는 것은 shader다. shader는 입력과 출력이 존재하며, 실행 과정에서 참조해야 하는 데이터가 별도로 존재한다.
### Input - Polygon Mesh
렌더링의 대상이 될 3D 모델을 구성하는 데이터. vertex와 이를 어떤 순서로 연결할 것인지를 결정하는 index로 구성된다.

파이프라인에 입력으로 공급되는 데이터 외에도 필요에 따라서 참조하는 여러 데이터가 존재한다.
### Output - Image
Display에 출력할 이미지. 2D 픽셀 배열.

