---
tags:
  - GameEngine
Date: 2026-01-10
---
# Overview
게임(혹은 그래픽)엔진의 그래픽 api에 대한 추상화 interface, 실제 구현에 해당하는 백엔드는 특정 그래픽 api(DirectX, OpenGL, Vulkan, Metal, etc)로 구성된다.
# Content
## Description
현대 그래픽 프로그래밍 환경은 그 실행 환경에 따라서 매우 파편화 되어있다. 게임 엔진의 경우, 이러한 파편화를 극복하기 위해서 그래픽 api를 엔진의 로직과 분리할 필요가 있는데, 이를 위한 추상화 레이어를 Render Hardware Interface, Rhi라 한다. 