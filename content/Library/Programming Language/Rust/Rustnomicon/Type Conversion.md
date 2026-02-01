---
Date: 2025-08-17
tags:
  - ProgrammingLanguage
---
# Overview
Rust에서 정의하는 타입 사이의 변환에 대해서 다룬다.
# Problem - 왜 Type Conversion이 unsafe한가?
## 일반적인 경우
결국 모든 데이터는 어딘가에 있는 비트 덩어리일 뿐이고, 타입 시스템은 우리가 그 비트들을 잘 쓰게 하기 위해 존재하는 메타데이터다. 

일반적인 프로그래밍 언어에서 타입이 제공하는 정보에는 size와 data layout이 있다. 즉, 해당 데이터의 크기와, 그 데이터를 구성하는 field에 접근하기 위해서 얼마나 offset을 주어야 하는 지가 바로 그것이다. 따라서 이론적인 관점에서 서로 다른 두 타입의 데이터 레이아웃이 같다면, 둘은 그대로 타입 간 변환이 가능하다.  

## Case 1 - ABI
러스트는 다른 프로그래밍 언어와는 달라서, 두 가지 측면에서 타입 간 변환이 곤란하다. 하나는 rust 특유의 최적화로 인한 layout 변화이다. rust는 ABI가 고정되지 않은 탓에 어느 타입을 정의하는 필드의 구성이 같다고 하더라도, 이들 둘 사이의 변환을 보장하지 않는다.
``` rust
// from Rustnomicon, Type Conversion
struct Foo {
    x: u32,
    y: u16,
}

struct Bar {
    a: u32,
    b: u16,
}
```
위 예제에서 `Foo`와 `Bar`는 동일한 구성의 필드를 갖는다. 하지만, 이 두 타입의 layout은 같다고 보장할 수 없다.  

이러한 경우, 가장 간단한 해법은 다음과 같은 코드를 바탕으로 `into`, `From` 트레잇을 구성하는 것이다.
``` rust
// from Rustnomicon, Type Conversion
fn reinterpret(foo: Foo) -> Bar {
    let Foo { x, y } = foo;
    Bar { a: x, b: y }
}
```
## Case 2 - Lifetime
다른 한 문제는 life time이다. rust의 타입에는 size와 layout에 더해서 lifetime이 포함된다. 따라서 lifetime이 존재하는 한, 타입 변환은 이 정보를 손상시키게 된다. 이들은 매우 자주 발생하는 문제들이고, 따라서 러스트는 이런 종류의 문제들을 해결하는 몇 가지 방법들을 제공한다.
# Type Coercion
강제 변환은 두 타입 사이가 언어가 규정하는 특별한 관계에 있어, 별도의 구현 없이 자동으로 변환을 강제할 수 있는 경우를 말한다. 이러한 강제 변환에는 다음과 같은 경우가 존재한다.

- https://doc.rust-lang.org/reference/type-coercions.html#coercion-types

다만, 여기에도 주의할 사항이 있는데, 바로 강제 변환이 가능한 관계라 하더라도, 둘은 엄격하게 서로 다른 타입이라는 것이다. 그 예시가 바로 다음과 같다.
``` rust
// from Rustnomicon, Type Coercion
trait Trait {}

fn foo<X: Trait>(t: X) {}

impl<'a> Trait for &'a i32 {}

fn main() {
    let t: &mut i32 = &mut 0;
    foo(t);
}
```
위 예제에서 우리는 `&a i32`타입에 대해서 `Trait`을 구현했다. Rust에서 정의된 바에 따라 `&mut i32` 와 `&i32`는 강제 변환이 가능하다. 그러나, 둘은 엄격하게 서로 다른 타입이므로, `Trait`은 `&mut i32`에 대해서는 구현되지 않으며, `foo`를 호출할 수 없다.

# Type Casting
캐스팅은 한 타입의 데이터를 다른 타입의 데이터로 취급하는 것으로, Coercion을 포함하는 개념이다. Rust에서 Casting은 `as` 연산자를 통해서 명시적으로만 수행된다. 
``` rust
// (instance of Type1) 'as' Type2
let var1 : i32 = 44;
let var2 : f32 = var1 as f32;
```

C/C++와 달리, Rust에서 Casting은 다음과 같은 경우에 대해서 제한적으로 가능하다. 
- 각종 정수/실수 형태의 primitive type.
- 정수-포인터
- 참조자-포인터
- 포인터-포인터

여러 생성자 혹은 형변환 연산자를 호출하거나 동적 바인딩을 일으키는 C/C++의 캐스팅과 달리, Rust의 캐스팅은 위 경우가 끝이며, **절대 실패하지 않는다**. 캐스팅이 불가능한 경우는 그냥 컴파일이 안된다.  

위 경우에서 봤을 때, C와의 호환성을 위해서 포인터 관련 캐스팅은 여전히 가능함을 확인할 수 있다. 이는 **위험한 행동이지만 unsafe는 아닌데**, 그 이유는 **포인터의 역참조가 이미 unsafe**하기 때문이다. 

캐스팅 과정에서 데이터의 손실 등은 Rust에서도 여전히 발생할 수 있다.

# Transmute - reinterpret_cast
C++에 `reinterpret_cast`가 존재한다면, Rust에는 `Transmute`가 존재한다. transmute는 새로운 데이터의 생성이 아닌, 기존의 byte 데이터에 대한 재해석을 의미한다.

C++에서 reinterpret_cast는 low-level한 구현에서 유용하게 사용된다. 그러나 Rust에서 transmute는 그 취급이 다르다. **Rust에서 transmute는 핵폭탄이다.**
## 동작 방식
transmute는 함수로, 다음과 같은 시그니쳐를 갖는다.
``` rust
pub const unsafe fn transmute<Src, Dst>(src: Src) -> Dst
```
`Src`타입의 데이터 src의 소유권을 받아서 `Dst`타입의 인스턴스를 반환한다. 일견 단순한 `into`처럼 보일 수 있지만, 앞서 말한 바와 같이 bit-wise 재해석으로 구현되었다. 두 타입 사이의 유일한 제한 조건은 Size가 같아야 한다는 것이다.
## 위험성
이를 사용하여 여러 파멸적인 결과를 얻을 수 있는데, 대표적인 사례는 다음과 같다.
- `& Type` -> `&mut Type` : mutability를 생성하여 Rust의 borrow checker를 망가뜨린다.
- `&'a Type` -> `& Type` : Lifetime을 제거하여 무한 수명 참조자를 생성, borrow checker를 망가뜨린다.

뿐만 아니라, data field를 필요에 따라서 재구성하는 Rust의 특성 상, 거의 모든 경우의 transmute는 Undefined Behavior로 이어진다. 그나마 transmute의 안전한 사용이 가능한 경우, 즉 완벽하게 예측 가능한 경우의 동작은 ABI가 결정되어 예측 가능한 경우이다. 즉, `repr(C)`, `repr(transparent)`인 타입에 대해서 가능하다.

C++의 reinterpret_cast가 pointer를 대상으로 동작하여 읽는 방식을 바꾸는 구조였다면, transmute는 Rust의 소유권 개념으로 인해서 인스턴스 자체를 변경한다. 
# Reference
- https://doc.rust-lang.org/nomicon/conversions.html