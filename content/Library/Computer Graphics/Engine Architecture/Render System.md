---
tags:
  - GameEngine
Date: 2026-01-10
---
# Overview
렌더링 엔진의 기능의 모듈화.
# Content
## Description
Renderer를 구성하는 기능의 단위를 표현하는 객체. 기존에 구현한 [[Render Hardware Interface|Rhi]]를 활용하여 실제 렌더링 관련 동작을 수행한다.

주로 [[Render Dependency Graph|RDG]]를 구성하는 각 pass node를 생성하는 factory로 기능한다. 