---
Date:
tags:
  - ARM
---
# Overview
ARM 프로세서의 여러 캐시 정책에 대해서 간단히 정리한다.
# Contents
## 정책 요약 비교

| 방식                   | index           | tag                          | 장점                                                  | 문제                                |
| -------------------- | --------------- | ---------------------------- | --------------------------------------------------- | --------------------------------- |
| **PIPT**             | 물리 주소           | 물리 주소                        | aliasing 없음, 단순함                                    | TLB로 변환 후 캐시 접근해야 해서 느릴 수 있음      |
| **VIVT**             | 가상 주소           | 가상 주소                        | 빠름                                                  | aliasing, context switch flush 문제 |
| **VIPT_NONALIASING** | 가상 주소           | 물리 주소                        | 빠르고 aliasing 없음                                     | 캐시 크기/associativity/page size 제약  |
| **VIPT_ALIASING**    | 가상 주소           | 물리 주소                        | TLB와 캐시 접근 병렬화 가능                                   | synonym aliasing 관리 필요            |
| **ASID-tagged**      | 보통 VA 기반 구조와 결합 | tag에 ASID 포함, 프로세스별 가상 주소 구분 | 서로 다른 process의 aliasing 감소, context switch flush 감소 | synonym aliasing 자체는 해결 못 함       |
## cacheid - index와 tag의 매핑

- index: 프로세서가 캐시를 참조할 때 쓰는 주소, 가상/물리 구분
- tag: 캐시가 가리키는 메인 메모리의 주소, 가상/물리 구분

``` text
주소 = [ tag | index | block offset ]
```

> cache 내에서 index로 참조, set을 획득

> 획득한 set이 cache hit인지(내가 원하는 결과인지)는 tag 비교

> cache에 접근할 때, 물리/가상 tag? 물리/가상 index?

---
## 물리? 가상?

- **ram은 물리 주소로 접근**
- **프로세서는 가상 주로를 가짐**
- 주소 변환을 위해서 TLB 변환이 필요 -> **시간 소모**

> tag와 index를 physical/virtual로 caching 하느냐에 따라서 결정됨

---

## aliasing 문제

> virtual index에서만 발생

- 서로 다른 virtual address가 같은 physical address를 참조
- virtual index를 사용하면 cache 내부에 서로 다른 cache set으로 저장 가능
- cache 내에 **동일한 copy가 중복**으로 존재 -> 효율 감소
## aliasing 해결 방법

> non-aliasing이 되기 위해서는 index 비트가 page offset에 들어가야함

virtual address는 page 단위로만 수행되기 때문

---
# Summary

# Reference