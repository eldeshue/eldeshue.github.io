---
Date: 2026-05-17
tags:
  - Linux
  - ProgrammingLanguage
---
# Overview
리눅스 커널에서 발견한 흥미로운 매크로에 대해서 소개한다.
# Contents
## offsetof
특정 타입과 그 타입의 필드 변수를 가지고 **해당 멤버의 오프셋**을 계산하는 매크로이다. 

이 매크로가 유용한 이유는 바이트 패딩 및 코드 난독화를 통해 offset이 뒤엉킬 경우에도 동작하기 때문이다.
``` C
#define offsetof(TYPE, MEMBER) ((size_t)&(((TYPE *)0)->MEMBER))
```
동작 방식을 설명하면 대략 다음과 같다.

1. 값 0을 포인터로 삼고, 이를 TYPE이라는 구조체의 포인터로 캐스팅한다.
2. 이 포인터를 가지고 MEMBER를 역참조(`->`)한다. 이 때 포인터의 offset이동이 수행된다.
3. 이 역참조된 MEMBER를 다시 참조(`&`)하여 MEMBER로 이동된 주솟값을 획득한다.
4. 이 포인터를 `size_t`로 캐스팅하여 단순 offset으로 바꾼다.

놀랍게도 0이라는 리터럴을 포인터로 취급, 캐스팅할 수 있다는 생각이 대단히 놀랍다.
다만, 아무래도 좀 테크닉성이 짙다 보니 대부분의 경우 컴파일러 인트린식으로 정의된다.
## container_of
**타입, 필드 이름, 필드의 주소로 원본 구조체의 주소를 역산**하는 매크로. 정의는 다음과 같다.

``` C
#define container_of(ptr, type, member) \
    ((type *)((char *)(ptr) - offsetof(type, member)))
``` 

앞서 살펴본 offsetof 매크로를 활용, 현재 필드의 주소에서 오프셋을 빼서 원본 구조체의 시작 위치, 해당 타입의 포인터를 구한다.
# Summary

# Reference