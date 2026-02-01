---
Date: 2025-08-08
tags:
  - ProgrammingLanguage
---
# Overview
C/C++와 유사하게 Rust또한 low-levle에 접근이 가능한 언어이다. 따라서, 데이터 레이아웃에 대해서 조작할 수 있는데, 이들에 대한 내용은 거의 모두 unsafe하다. 

rust에서 어떤 custom data type에 대한 memory layout을 조작할 때, 주의해야 할 내용을 다룬다.
# repr
`repr`은 데이터 레이아웃을 의미하는 rust의 지시자이다. 흔히 다음과 같이 사용된다.
``` Rust

#[repr(KeyWord)]
struct MyData {
	// ...
}
```
여기서 `KeyWord`에 어떤 값을 적는 지에 따라서 `MyData`의 layout이 결정된다.
# repr(Rust) - Rust의 Layout
기본적으로 rust의 모든 데이터는 byte단위로 저장된다. 이를 바탕으로 addressing이 동작하기 때문이다. 일부 특수한 경우(SIMD 등)에서는 2의 거듭제곱을 기준으로 align되기도 한다.

> **어떤 데이터가 n byte로 align되었다는 것은 해당 데이터의 시작 주소가 항상 n의 배수임을 의미한다.**

Rust에서 데이터는 다음의 두 원칙을 따라서 alignment를 갖는다.
- primitive type : 자신의 크기로 align된다. 
	- ex) i32는 4byte, i64는 8byte로 align 된다.
- complex type : 자신이 갖는 필드 중 가장 큰 사이즈를 갖는 primitive type의 alignment를 갖는다.

Rust는 다음과 같은 타입으로 복합 데이터 타입이 존재한다.

- struct(명명된 제품 유형)
- tuple(익명의 제품 유형)
- array(동질 제품 유형)
- enum(tagged union, variant in c++)
- union(tagless)

**Rust의 ABI는 지속적인 발전 과정에 있으므로, 그 명세가 공개되지 않는다. 따라서 rust의 abi를 직접적으로 이용하는 것은 옳지 않다.**
## Exotic Type in Rust
rust 고유의 type에 대해서 다룬다.
### Dynamic Sized Type
컴파일 타임에 그 크기가 명시되지 않는 데이터의 타입. C에 비유하자면, malloc으로 동적 할당한 memory block을 위한 타입이라 할 수 있다.

Rust에서는 두 가지 대표적인 DST가 존재한다.
- Trait Object : `dyn Trait`, Rust 방식의 dynamic binding. virtual table 활용.
- Slice : `[T]`, 앞서 예로 들었던 malloc된 heap allocated된 memory block.
두 타입 모두 참조의 형태로 접근하는 공통점이 있는데, 이는 해당 타입의 실제 크기가 dynamic하기 때문이다.

> **DST는 size를 알 수 없기 때문에 참조의 형태로만 사용되어야 한다.**

[Rustnomicon](https://doc.rust-lang.org/nomicon/exotic-sizes.html)에 의하면 DST 자체로는 큰 의미가 없으며, 다음과 같은 방식으로 upcasting할 수 있음이 알려져 있다.
``` Rust
struct MySuperSliceable<T: ?Sized> { // ?Sized 라는 표현이 T가 DST임을 의미함
    info: u32,
    data: T, // DST인 필드는 struct의 마지막에만 위치할 수 있다. 어찌보면 당연하다.
}

fn main() {
    let sized: MySuperSliceable<[u8; 8]> = MySuperSliceable {
        info: 17,
        data: [0; 8],
    };

    let dynamic: &MySuperSliceable<[u8]> = &sized; // upcasting 수행

    // prints: "17 [0, 0, 0, 0, 0, 0, 0, 0]"
    println!("{} {:?}", dynamic.info, &dynamic.data);
}
```
이러한 upcasting 형태의 응용은 흥미롭지만, 그다지 유용하지는 않다.
### Zero Sized Type
Rust에는 marker를 비롯하여 사상적으로만 존재하고, 메모리를 점유하지 않는, zero sized type이 여럿 존재한다. 

ZST에는 다음과 같은 경우가 있다.
``` Rust
// C와는 다르게 선언과 구현의 구분이라는 개념이 없음.
// 필드가 존재하지 않으므로, 크기는 0임.
struct ZeroSizedType1;

struct ZeroSizedType2 {
	qux: (), // 빈 튜플, zst
	[u8; 0], // 빈 배열, zst
}
```
 Rust는 ZST를 생성하거나 저장하는 모든 연산을 무(no-op)로 줄일 수 있으며, 이를 저장하는 것 자체가 의미가 없다. 따라서, 언제 어디서나 생성할 수 있고, 또 소멸할 수 있다. 그러나, ZST에 대한 참조는 다음의 특별 규칙을 따르는데, 상당히 흥미롭다.

> **ZST에 대한 참조는 그 참조하는 대상이 null이 아니어야 한다. 하지만, null을 읽어서 zst를 생성하는 것은 UB가 아니다.**

zst를 null에서 생성할 수 있다는 점이 흥미로운데, 이는 zst에 대한 접근이 실제 operation으로 이어지지 않기 때문이다. null 주소에 접근하여 읽으려 한다 해도, **실제로 메모리에 접근하는 바가 없으니**, seg fault가 발생할 수 없다.
# repr(C) - C의 Layout
Rust의 ABI는 고정되지 않았다. 따라서, 만일 다른 언어에서 rust의 코드를 사용하고자 한다면, Data의 layout 불일치로 인해서 상당히 곤란한 문제가 발생한다. 이를 위해 존재하는 부분이 바로 `repr(C)`, C의 ABI를 따르는 것이다. 즉, rust의 데이터를 정확하게 C의 Layout을 갖도록 유도하여 rust 코드를 마치 C처럼 사용할 수 있도록 하는 것이다. 이는 C++의 `extern "C"`와 동일한 효과를 갖는다.

> **repr(C)를 이용하여 Rust 코드를 C compatible하게 사용할 수 있다.**

이를 통해서 빌드된 바이너리는 C나 C++에서 사용할 수 있다. 이러한 이종의 코드를 혼합하는 것을 FFI라 하는데, 여기에도 여전히 주의할 부분이 있다.
# repr(transparent)
0이 아닌 단일 필드(크기가 0인 필드가 더 있을 수 있음)를 갖는 구조체 또는 단일 변형 열거형에만 사용할 수 있다. 전체 구조체/열거형의 레이아웃과 ABI가 해당 필드와 동일하게 보장된다.

> **transparency는 `wrapper<T>` 타입의 layout을 그 원소로 갖는 단일 필드, `T`와 동일하게 유지하는 것이다.**
 
이러한 transparency의 목적은 transmute 등을 통한 두 타입 사이의 low level conversion이다.

