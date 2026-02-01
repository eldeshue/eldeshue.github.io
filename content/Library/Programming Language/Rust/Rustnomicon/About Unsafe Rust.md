---
Date: 2025-08-08
tags:
  - ProgrammingLanguage
---
# Unsafe Rust란 무엇인가?

> Rust can be thought of as a combination of two programming languages: _Safe Rust_ and _Unsafe Rust_. Conveniently, these names mean exactly what they say: Safe Rust is Safe. Unsafe Rust is, well, not. In fact, Unsafe Rust lets us do some _really_ unsafe things. Things the Rust authors will implore you not to do, but we'll do anyway.

Rust에는 두 가지 측면이 있다. 하나는 강력한 type system에 기반하여 정적 분석기의 도움을 받는, borrow checker와 lifetime의 보호를 받는, safe한 rust이다. 

다른 하나는 정적 분석기의 보호를 받지 못하는 rust로, 소위 unsafe rust이다. 이 unsafe rust를 정상적으로 잘 사용하기 위해서는 알아야 할 여러 중요한 사항이 있는데, 이 내용은 매우 까다롭다. rust의 철학이 safety에 있음을 생각하면, unsafe rust의 존재는 조금 이질적일 수 있지만, 단순 구현이 아닌, 여러 utility의 개발을 위해서는 unsafe rust의 존재가 매우 필수적이라는 점을 알 수 있다. 

Rustnomicon은 이러한 unsafe rust에 대해서 다루는 일종의 guide book이다. 이하 모든 내용은 해당 [사이트](https://doc.rust-lang.org/nomicon/intro.html)에 기원한다.
# Unsafe Rust has to be safe.

> **Unsafe Rust는 Safe해야 한다. 그저, 언어가 제공하는 safety feature의 도움을 받을 수 없을 뿐이다.**

unsafe rust란 본질적으로 rust의 safety로 추상화 된 low level 구현의 영역이다. 상당히 많은 rust의 safety관련 기능들은 unsafe rust에 의존하고 있으며, 이 경우 여전히 safety를 보장해야 한다. **즉, safety의 책임이 언어에서 프로그래머로 이동한 것이다.**

> **Unsafe Rust에서 safety의 책임은 프로그래머에게 있다.**

rustnomicon에서는 unsafe rust를 사용하면서 어떻게 safe한 결과물을 얻을 것인지에 대해서 다룰 것이다.

# 신뢰성 원칙 - Do not trust Safe Rust
unsafe rust를 사용함에 있어 중요하게 여겨져야 할 원칙이 있는데, 그것은 코드에 대한 신뢰 여부이다.

먼저, safe rust는 언어 구현 상, 컴파일이 되었다면, UB가 없음을 언어가 보장한다. 따라서 어떤 구현이 의존하는 대상이 safe하고, 내가 작성한 코드가 safe하다면(문제없이 컴파일 되었다면), 나의 코드는 safe하다. 따라서 safe rust는 믿을 수 있다. **이는 safe rust사이의 신뢰이다.**  

unsafe rust는 safety 추상화가 적용되지 않는 low level 구현이다. 따라서, unsafe rust에 의한 구현은 또 다른 unsafe에 의존할 확률이 대단히 높다. **따라서 unsafe rust사이에서는 신뢰가 없다.**

그렇다면, unsafe rust는 safe rust를 믿어야 할까? 답은 아니오다. unsafe rust의 구현 책임은 프로그래머에게 있다. 따라서, unsafe 구현이 safe한 구현에 의존한다고 하더라도, 프로그래머는 이 safe한 구현의 오작동 여부에 대해서 방어할 수 있어야 한다.

> **Unsafe Rust의 구현 책임은 프로그래머에게 있다. 따라서 unsafe 구현의 safety는 의존하는 구현이 아닌, 프로그래머가 보장해야 한다.**

# Unsafe Rust의 범위
다음의 구현은 unsafe하다고 본다.
- raw pointer의 dereference
- unsafe한 trait, 함수 등의 사용
- static한 데이터에 대한 수정(가변 참조 접근 등)
- union의 필드 접근(transmute)

위 기능의 사용은 일반적으로 UB로 이어질 수 있다. 다음은 rust의 UB이다.

- ( `*`연산자 on을 사용하여) dangling 포인터나 정렬되지 않은 포인터의 역참조(아래 참조)
- 포인터 [별칭 규칙 위반](https://doc.rust-lang.org/nomicon/references.html)
- 잘못된 호출 ABI로 함수를 호출하거나 잘못된 unwind ABI로 함수에서 해제.
- [data race](https://doc.rust-lang.org/nomicon/races.html) 유발[](https://doc.rust-lang.org/nomicon/races.html)
- 현재 실행 스레드가 지원하지 않는 [대상 기능](https://doc.rust-lang.org/reference/attributes/codegen.html#the-target_feature-attribute) 으로 컴파일된 코드 실행
- `enum`잘못된 값 생성(단독으로 또는 / `struct`/array/tuple 과 같은 복합 유형의 필드로 ):
    - `bool`0이나 1이 아닌 a : C와 달리 rust는 0/1만 bool로 판단함.
    - `enum`매칭 실패
    - 유효하지 않은 `fn`포인터
    - `char`[0x0, 0xD7FF] 및 [0xE000, 0x10FFFF] 범위를 벗어남
    - a `!`(이 유형에는 모든 값이 유효하지 않습니다, never type)
    - [초기화되지 않은 메모리](https://doc.rust-lang.org/nomicon/uninitialized.html) 에서 읽은 정수( `i*`/ `u*`), 부동 소수점 값( ), 원시 포인터 또는 .의 초기화되지 않은 메모리 .`f*`[](https://doc.rust-lang.org/nomicon/uninitialized.html)`str`
    - 스마트 포인터 `Box`의 불안정하거나, 정렬되지 않았거나, 잘못된 값에 대한 참조.
    - 잘못된 메타데이터가 있는 참조, `Box`또는 원시 포인터:
        - `dyn Trait`, `Trait`포인터가 실제 동적 특성과 일치하는 vtable에 대한 포인터가 아니면 메타데이터는 유효하지 않습니다.
        - 길이가 유효하지 않으면 슬라이스 메타데이터는 유효하지 않습니다 `usize` (즉, 초기화되지 않은 메모리에서 읽어서는 안 됨).
    - 사용자 지정 유효하지 않은 값을 갖는 유형은 [`NonNull`](https://doc.rust-lang.org/std/ptr/struct.NonNull.html)null인 a와 같은 값 중 하나입니다. (사용자 지정 유효하지 않은 값을 요청하는 기능은 불안정하지만, 와 같은 일부 안정적인 libstd 유형은 `NonNull`이를 활용합니다.)

