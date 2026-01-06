---
tags:
  - Vulkan
---
# Overview
shader를 source of truth로 삼는 shader reflection에 대해서 알아보자
# Problem
## 1. 레이아웃 조합 폭발
핵심 문제는 **렌더러는 컴파일 타임에 shader의 layout을 알 방법이 없다**는 것이다. 

그 결과, 렌더러는 입력될 것으로 예상되는 조합의 layout을 미리 준비하여야 하는데, 이는 조합 폭발로 인해서 현실적이지 않다.

이는 범용성을 유지해야 하는 게임 엔진의 경우, pso 관리의 어려움으로 이어진다.

## 2. 레이아웃 불일치
pipeline layout과 shader의 layout이 불일치하는 경우, 미정의 동작(UB)로 이어진다.

정상적인 렌더링이 불가능하며, 심한 경우 데이터 오염과 렌더링 실패로 이어질 수 있다.

> validation layer가 활성화된 디버그 빌드에서는 렌더링이 실패한다.

# Solution - Shader Reflection
이러한 binding 문제를 해결하기 위해서 shader reflection이라는 기능이 개발되었다.

shader reflection은 컴파일 된 shader binary인 spir-v를 읽어들여 pipeline layout을 추출하는 기능이다.

이를 통해서 **shader를 런타임에 읽어서 binding 정보를 획득하고, 그에 맞는 pso를 구성할 수 있다**.

대표적인 구현체로는 크로노스 재단의 **SPIRV-Cross**가 있다.