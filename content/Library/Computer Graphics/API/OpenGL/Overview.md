---
Date: 2025-03-17
tags:
  - Graphics
  - OpenGL
---
# About OpenGL
## What is OpenGL?
OpenGL(이하 OGL)은 흔히 graphics api라 생각되지만, 사실은 일종의 규격이다. api를 관리하는 크로노스 그룹은 OpenGL의 구현 명세만 정의하고, 여러 GPU 제조사가 이 명세를 따르는 실제 api를 구현한다. 따라서 구현체에 따라 일부 동작이 상이할 수 있다.

## Immediate Mode VS Core-profile Mode
OGL에는 크게 두 종류의 모드가 있으며, 각 모드마다 여러 함수(api)가 존재한다.

Immediate mode는 오래된 api로, 높은 추상화를 특징으로 한다. 이 추상화로 인해서 programmability가 낮고, 상당히 비효율적이지만, 사용이 쉽다는 장점이 있다.

Core-profile mode는 기존의 api가 가진 문제를 극복하기 위해서 등장한 새로운 api로, 추상화의 정도는 낮지만, 높은 programmability로 인해서 효율적인 코드를 작성할 수 있다. 

Immediate mode의 api들은 상당수가 deprecated되어 사용하면 error가 발생할 수 있다.

## State Machine
OpenGL은 다수의 변수가 모여서 정의되는 거대한 상태기계(State Machine)이며, 이를 흔히 context라 부른다. OpenGL을 통한 렌더링은 이 상태기계의 상태를 전환하는 것을 통해서 이루어지며, 이러한 상태(Context)는 global하게 존재하기 때문에 병렬성에 있어서 일부 손해를 본다.

이러한 구조는 api의 사용 전반에 영향을 미쳐서, 상당수의 함수들이 특정 대상에 대한 focus를 두는 구로조 작동한다. 이러한 focus를 옮기는 류의 함수들은 bind라는 이름을 갖는다. 이러한 단일 context는 직관적인 코드를 제공하지만, 멀티 스레딩을 비롯한 복잡한 구조에서는 오히려 구조적 제약으로 작용한다.

DirectX의 구조는 Component Object Model이라 하는 객체지향 방법론을 따른다. 이는 렌더링을 하기 위한 자원과 파이프라인의 상태를 별도로 관리한다는 뜻이며, 렌더링 자원이 상태와 분리되어 별도로 관리할 수 있음을 의미한다. 

즉 요약하자면, OpenGL에서는 자원의 변화나 파이프라인 상태의 변경이 모두 상태 기계의 상태로 존재하며, 이들 변화를 api함수의 실행으로 순차적으로 적용하고, 렌더링을 실시한다. 그러나 DirecX는 상태와 자원이 분리되어, 상태의 변화와 자원의 변화를 따로(혹은 병렬로) 수행할 수 있다.
### Object and Binding
앞서 설명한 state machine의 상태를 변경하기 위해서 흔히 object를 사용한다. object는 일종의 구조체로, 여러 상태에 대한 정보를 정의한다. opengl에서는 이러한 상태를 조작하기 위해서 이들 object를 bind하는 구조를 많이 사용한다. 

이러한 binding은 DirectX도 일부 공유하는데, 다만 DirectX에서는 보다 명시적으로 이루어진다.

object와 bind 구조의 장점은 일종의 lazy evaluation에 있다. 즉, 미리 object를 생성하고, 옵션을 설정했다면, 필요한 순간에 bind만 수행하여 빠르게 상태를 변경할 수 있음에 있다.

# Summary
OpenGL의 개요를 정리했다. 이전에 조금 공부하다 포기했던 DirectX12에 비하면 확실히 쉽다고 생각된다. 하지만, 이 쉽다는 점이 진짜 쉬워서 쉽다기 보다는, GPU를 상태기계로 모델링한 탓이라 추후에 비싼 댓가를 치루리라는 예감이 든다. 

어째서 DirectX12에서 그토록 복잡한 객체 구조를 갖는지 알게 되는 부분이다. 그토록 거대하고 복잡한 gpu를 객체로 추상화 하자니 별 수가 없었으리라. 

다만, DirectX에 비하면 COM을 비롯한 낯선 구조가 없어서 조금은 쉽게 접근할 수 있다고 생각한다.