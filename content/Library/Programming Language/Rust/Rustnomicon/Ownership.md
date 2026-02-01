---
Date: 2025-08-08
tags:
  - ProgrammingLanguage
---
# Overview
Rust의 핵심 기능인 Ownership과 그에 관련된 Rustnomicon의 여러 내용을 다룬다.

이 글에서 사용하는 모든 예제는 Rustnomicon에서 가져왔거나 일부 변형한 것이다.
# Aliasing
Rust에서는 다음과 같은 경우에 대하여 aliased되었다고 본다.
``` rust
let data : SomeType = SomeType::new();
let ref = &data;
let aliased = ref;    /// ref의 alias 
```
alias는 같은 대상을 가리키는 참조를 의미한다. 이는 참조라는 것이 본질적으로는 변수로 표현되는 한 데이터에 대해 접근할 수 있는 변수를 추가하는 것이기 때문이다. 따라서 같은 대상을 가리키는 여러 참조는 해당 원본에 대한 별명(alias)이다.

alias에는 다음과 같은 원칙이 있다.

- 모든 참조는 원본보다 오래 살 수 없다(cannot outlive).
- `&T`는 여럿 존재할 수 있다.
- `&mut T`는 오직 하나만 존재할 수 있다.
- `&mut T`는 `&T`와 양립할 수 없다.

> **`&mut T`가 없다면, 자유롭게 alias할 수 있다.** 

Rust는 이러한 위 원칙을 바탕으로 alias를 추적, 관리하는데, 그 목적은 다음과 같다.

- 값의 메모리를 접근하는 포인터가 없다는 것을 증명함으로써 값들을 레지스터에 그대로 두는 것
	-> mut ref가 ref와 공존하는 경우에만 가능, mut ref가 없으니 coherence가 보장됨.
- 어떤 메모리는 마지막으로 읽은 후에 쓴 적이 없다는 것을 증명함으로써 읽기 작업들을 제거하는 것
	-> 마찬가지로 mut ref가 없으니 coherence가 보장됨. 캐싱된 데이터 재활용 가능
- 어떤 메모리는 다음 쓰기 작업 전에 읽은 적이 없다는 것을 증명함으로써 쓰기 작업들을 제거하는 것
	-> 중간에 read가 없으므로, write back 가능
- 읽기 작업들이나 쓰기 작업들이 서로에 의존하지 않는다는 것을 증명함으로써 작업들을 옭기거나 순서를 바꾸는 것
	-> mut ref의 작용을 기점으로, 의존성이 명확히 구분됨. 따라서 재배치 가능함.

이러한 최적화는 여타의 언어에서는 제한적으로 가능한 영역이었으나, Rust에서는 확신을 가지고 수행할 수 있다.

# Lifetime
Rust의 근간을 이루는 여러 법칙은 결국 이 lifetime이라는 개념 위에 존재한다. lifetime의 정의는 다음과 같다.

> **lifetime이란 어느 reference의 유효한 영역을 정의한 것이다. 이 영역은 불연속, 비선형일 수 있다.**

불연속은 re-borrow때문이며, 비선형성은 분기 때문에 발생한다.
## Local Lifetime의 시각화
Rustnomicon에 따르면 수명은 다음과 같은 방식으로 시각화 할 수 있다. 다음은 여러 한 변수에 대해서, 이를 참조하는 여러 참조의 생성에 관한 예제이다.
``` rust
// from rustnomicon, simple let statement
let x = 0;
let y = &x;
let z = &y;
```
위 예제에 대하여 수명을 시각화 하면 다음과 같다.
``` rust
// from rustnomicon, visualization of lifetime
// 주의: `'a: {` 나 `&'b x` 는 유효한 문법이 아닙니다!
'a: {
    let x: i32 = 0;
    'b: {
        let y: &'b i32 = &'b x;
        'c: {
            let z: &'c &'b i32 = &'c y; // "i32의 레퍼런스의 레퍼런스"
        }
    }
}
```
위 예제는 let 선언문이 lifetime을 동반하는 어떤 scope를 암묵적으로 생성함을 보여준다.

## 함수의 Lifetime 시각화
다음의 `as_str` 함수는 한 `u32`변수를 format으로 가공하여 `&str`로 반환한다.
``` rust
fn as_str(data: &u32) -> &str {
    let s = format!("{}", data);
    &s
}
```
이 함수에 lifetime을 표시하면 다음과 같다.
``` rust
fn as_str<'a>(data: &'a u32) -> &'a str {
    'b: {
        let s = format!("{}", data);
        return &'a s;
    }
}
```
위 예제는 lifetime을 어긴 대표적인 예시다. 먼저 `format!`매크로는 `u32`를 가지고 `String`을 생성하며, 이를 가리키는 `&str`을 반환한다.

여기서 반환하는 `&str`인 `s`가 가라키는 `String`의 lifetime은 `'b`인데, 이는 `data`의 lifetime인 `'a`보다 짧다. 그러므로, `s`의 수명은 `'a`가 될 수 없다. 수명의 확장은 불법이다.
## lifetime의 끝 - 소멸 시점
기본적으로 수명은 해당 객체가 마지막으로 사용된 위치에서 끝난다.
``` rust
let mut data = vec![1, 2, 3];
let x = &data[0];
println!("{}", x);  // <- 참조 x의 마지막 사용, 여기서 x의 수명 종료

data.push(4); // 따라서 여기서 &mut data를 사용할 수 있음.
```
그러나, `Drop` 트레잇을 구현한 경우 그 lifetime은 scope의 종료까지 연장된다.
``` rust
#[derive(Debug)]
struct X<'a>(&'a i32);

impl Drop for X<'_> {
    fn drop(&mut self) {}
}

{
	let mut data = vec![1, 2, 3];
	let x = X(&data[0]);
	println!("{:?}", x);
	data.push(4);
	
	// Drop이 구현되었으므로, 여기서 x의 소멸자가 호출됨
	// push의 &mut data와 충돌함
}
```
## 비선형 lifetime
다음의 경우 분기에 따라서 참조 `x`의 마지막 사용 위치가 달라진다. 따라서 수명은 비선형이다.
``` rust
let mut data = vec![1, 2, 3];
let x = &data[0];

// some_condition()의 평가에 따라서 분기함
// 분기 결과에 따라서 x의 마지막 사용 위치가 달라짐
if some_condition() {
    println!("{}", x); // 이것이 이 가지에서 `x`의 마지막 사용
    data.push(4);      // 따라서 여기서 push할 수 있죠
} else {
    // 여기에는 `x`의 사용이 없으므로, 사실상 마지막 사용은
    // 이 예제 맨 위에서 x의 정의 시점입니다.
    data.push(5);
}
```
## 불연속 lifetime
다음의 경우 reborrow가 발생하여 lifetime이 끊기는 경우에 대한 예제다. 이 때 borrow 당한 참조의 유효성(lifetime)은 borrow한 참조가 소멸할 때 까지 일시적으로 정지된다. 따라서 lifetime은 불연속일 수 있다.
``` rust
let mut x = 10;

let r = &mut x;   // r: &mut i32 (x에 대한 배타적 접근권 확보)

let r2 = &*r;     // r을 통해 다시 &i32 (immutable ref) 생성 → reborrow
println!("읽기만 가능: {}", r2);

// *r += 1; // ❌ 불가능: r은 freeze 상태
// x += 1;  // ❌ 불가능: x도 freeze 상태 (원본 접근 불가)

// r2가 더 이상 사용되지 않으면 freeze 해제
*r += 1;          // ✅ 이제 다시 수정 가능
println!("변경 후: {}", r);
```
## 무제한(Unbounded) Lifetime
특정한 방법으로 얻어진 참조는 무제한 lifetime을 가질 수 있음. 무제한 수명은 `'static`과 유사하며, 얻어온 참조의 원본이 정상적이지 않은 경우에 해당한다. 대표적인 예가 다음과 같음.
``` rust
fn get_str<'a>(s: *const String) -> &'a str {
    unsafe { &*s } // pointer로 부터 얻어낸 참조
}

fn main() {
    let soon_dropped = String::from("hello");
    let dangling : &str = get_str(&soon_dropped);
    drop(soon_dropped);
    println!("잘못된 str: {}", dangling); // 이미 drop된 객체를 향한 참조
}
```
위 예제는 pointer를 역참조하여 reference를 얻어오는 예제로, pointer는 참조와 달리, 가리키는 원본 객체에 대한 lifetime 정보를 가지고 있지 않음. 그 결과 rust는 이 pointer에서 얻어낸 참조의 수명을 추론할 수 없으며, 이 경우 수명은 무한대(Unbounded)가 됨.

정상적인 활용을 위해서는 적절한 함수를 통한 lifetime의 제한이 필수적이다. 일반적으로, 반환값의 수명은 함수의 인자의 수명으로 제한되기 때문이다.

이러한 무제한 수명의 원인에는 `transmute`를 통한 참조의 변환도 가능함.
# HRTB - Trait에서 lifetime 표현
HRTB은 Higher Rank Trait Bound의 약자로, 기존의 타입에 대해서 Trait에 대한 제약만 거는 trait bound에 대하여 lifetime의 조건을 추가한 것이다. lifetime에 대하 제약을 거는 것은 보다 고등한(Higher Rank)한 것으로 보여서 이러한 이름을 얻은 것으로 보인다. 

HRTB는 callable을 만드는 다음과 같은 경우에 주로 사용된다.
``` rust
fn call_with_str<F>(f: F)
where
    F: Fn(&str) -> &str
{
    let s = String::from("hello");
    println!("{}", f(&s));
}
```
위 에제에서는 trait bound를 통해서 특정 트레잇을 구현한 어떠한 타입만 전달받을 수 있는 함수 제네릭을 구현했다. 그러나, 이 예제에서 보이듯이 Fn 트레잇은 그 정보가 충분하지 않은데, 여기에는 lifetime에 대한 정보가 결여되어 있다. f가 callable이고, 그 시그니쳐와 반환값 사이의 lifetime이 명시되지 않았다. 이러한 lifetime 정보를 명시하고자 한다면, 다음과 같이 HRTB, 즉 `for<'a>` 를 사용한다.
``` rust
fn call_with_str<F>(f: F)
where
    F: for<'a> Fn(&'a str) -> &'a str
{
    let s = String::from("hello");
    println!("{}", f(&s));
}
```
위의 HRTB를 통해서 개선된 `call_with_str`은 인자로 받는 f에 대하여, lifetime이 명시되어 있다.

> **HRTB를 통해서 trait bound에 lifetime에 대한 조건을 추가적으로 부여할 수 있다.**
# Drop Checker
drop checker는 컴파일러의 일부분으로, 어떤 타입의 `Drop`트레잇의 구현에 대하여, 안정성을 보장하기 위해 존재한다. drop checker에게는 하나의 원리가 존재하는데, 이는 다음과 같다.

> **Drop checker는 어떤 타입 A에 대하여, A의 필드로 있는 모든 참조의 원본이 A보다 오래 살도록 강제한다.**

이에 대한 예시는 다음과 같다.
``` rust
struct Inspector<'a>(&'a u8);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("I was only {} days from retirement!", self.0);
    }
}

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days));
    // `days`가 먼저 해제되게 되었다고 가정해 봅시다.
    // 그럼 `Inspector`가 해제될 때, 이미 해제된 메모리를 읽으려고 할 겁니다!
}
```
위 예제에서는 Drop checker의 존재 이유에 대해서 보여주고 있다. `World`의 필드인 `days`가 먼저 소멸하고, 이후 `inspector`가 소멸하여 그 소멸자가 호출된다고 하면, `Inspector<'a>`의 drop이 호출되며, 삭제된 days를 참조하게 된다. 이는 Rust가 허용할 수 없는 상황이다. 따라서, 이러한 상황을 예방하기 위해 `inspector`는 `days`보다 무조건 오래살도록 강제되어야 한다.

그러나 이 조건은 지나치게 엄격한 부분이 있는데, 다음 예제가 이를 잘 보여준다.
``` rust
struct Inspector<'a>(&'a u8, &'static str);

impl<'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {})는 보지 *않아야* 할 때를 압니다.", self.1);
    }
}

struct World<'a> {
    inspector: Option<Inspector<'a>>,
    days: Box<u8>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
    // `days`가 먼저 해제되게 된다고 해 봅시다.
    // `Inspector`가 해제되어도, 그 소멸자는 빌린 `days`를
    // 접근하지 않을 겁니다.
}
```
위 예제에서는 `Inspector<'a>`의 `Drop`이 `days`를 참조하지 않아서 dangling할 염려가 없다. 그러나 위 예제 또한 여전히 컴파일 되지 않는데, 그 이유는 컴파일러가 `Drop`의 구현에 대해서 이해하지 못하기 때문이다. 

따라서 Drop Checker는 그 건전성을 위해서 과하게 엄격하지만, 필드로 갖는 모든 참조의 수명이 그 자신보다 무조건 길도록 강제한다. 
## may_dangle
Drop Checker의 동작에 예외를 주기 위하여, 특정 lifetime 변수에 `may_dangle`속성을 부여할 수 있다. 이 속성이 부여된 lifetime을 갖는 변수는 drop checker의 참조 수명 검사에서 예외로 처리된다.
``` rust
#![feature(dropck_eyepatch)]

struct Inspector<'a>(&'a u8, &'static str);

// may_dangle 속성으로, Inspector에 대한 drop checker를 패스함
unsafe impl<#[may_dangle] 'a> Drop for Inspector<'a> {
    fn drop(&mut self) {
        println!("Inspector(_, {})는 보지 *않아야* 할 때를 압니다.", self.1);
    }
}

struct World<'a> {
    days: Box<u8>,
    inspector: Option<Inspector<'a>>,
}

fn main() {
    let mut world = World {
        inspector: None,
        days: Box::new(1),
    };
    world.inspector = Some(Inspector(&world.days, "gadget"));
}
```
위 예제는 `may_dangle`을 통해서 drop checker에게 `Inspcetor`의 문제를 무시하게끔 만들었다. 따라서 위 코드는 컴파일되며 실행이 가능하다. 단, `may_dangle`의 사용으로 drop checker의 작용을 무효화한 만큼, **그 안정성을 개발자가 보장해야만 한다.**

> **may_dangle은 문자 그대로 dangling 위험이 지대하므로, 사용자가 아주 확신할 수 있을 때에만 제한적으로 사용해야 마땅하다.**
# Phantom Data
`PhantomData`는 공간을 차지하지 않지만, 컴파일러의 분석을 위해 주어진 타입의 필드를 흉내내는 가상의 데이터다. 이를 통해서 자동 트레잇과 drop checker에 필요한 정보 등의 유용한  것들을 컴파일러에게 제공한다.

이러한 가상의 데이터가 필요한 이유 중 하나는 **타입에 lifetime이라는 중요한 메타정보가 포함**되기 때문이다. 그러나, 구현 상으로는 해당 정보를 표현할 방법이 없을 때, 이 `PhantomData`를 통해서 추가적인 정보를 넣어줄 수 있다.

대표적인 예시가 다음과 같은 이터레이터의 구현이다.
``` rust
use std::marker;

struct Iter<'a, T: 'a> {
    ptr: *const T,
    end: *const T,
    _marker: marker::PhantomData<&'a T>, // Iter가 T를 참조함을 표현한다.
}
```
이터레이터는 T타입의 원소를 명백하게 참조하고 있다. 그러나, 그 구현은 pointer로 되어 있어, 이러한 참조와 그에 따른 lifetime 정보를 표현할 방법이 없었다. 따라서, `PhantomData`를 도입하는 것으로, 해당 이터레이터가 어떤 데이터 T를 참조하며, 그 원본의 lifetime에 bound됨을 표현한다. 

내 생각에는 `PhantomData`의 도입은 필요에 의하기 보다는, **그냥 관용적으로 해당 구현이 그 타입의 데이터를 소유/참조한다면 그냥 넣는게 좋다**고 생각한다. 별도로 데이터를 소비하지 않기 때문에, 넣는다고 크게 손해볼 일도 없다.

> **PhantomData를 넣지 않아서 생기는 문제는 많지만, 넣는다고 생길 문제는 그다지 없다.**
# Reference
- https://doc.rust-lang.org/nomicon/lifetimes.html