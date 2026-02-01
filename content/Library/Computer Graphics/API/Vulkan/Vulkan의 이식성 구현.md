---
Date: 2025-10-16
tags:
  - Vulkan
---
# Description
Vulkan에서 이식성은 어떻게 구현되는가?

# Contents
## VK_KHR_portability_enumeration
이는 extension의 일종으로, physical device 중 portability를 구현한 하드웨어를 선택할 수 있도록 하는 기능을 추가한다.

여기서 말하는 portability는 native vulkan이 구현되지 않았음을 의미하는데, 대표적인 경우가 apple 생태계의 vulkan api이다.

apple은 metal이라는 고유한 graphic api를 가지고 있으며, vulkan을 native로 지원하지 않는다. 그러나 vulkan은 여전히 apple 제품에서 실행할 수 있는데, 그 이유는 vulkan을 metal 코드로 번역하는 moltenVK라는 변환 layer가 존재하기 때문이다. 그리고 이 molenVK은 바로 이 portability extension에 의해서 활성화된다.

