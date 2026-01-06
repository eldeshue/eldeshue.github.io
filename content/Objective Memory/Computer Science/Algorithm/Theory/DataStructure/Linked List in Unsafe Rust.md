---
Date: 2025-07-27
tags:
  - Algorithm
  - ProgrammingLanguage
---
# Overview
Linked List를 구현하면서 사용하게 된 Rust의 safety check와 이를 우회하는 unsafe한 feature에 대해서 알아보자.

해당 구현은 다음의 주소에서 확인할 수 있다.
> https://github.com/Yeongtong42/UnderthRust/tree/19-feat-chapter-10-linked-list/collections/list
# Implementation of List
링크드 리스트는 다음과 같은 형태로 구현된다.
``` Rust
pub struct List<T> {
    len: usize,
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
}
```
`List`는 T타입 데이터를 저장하는 `Node`를 관리하며, 이들은 heap에 할당되어 pointer의 형태로 관리된다. `List`는 여러 노드 중 맨 앞인 head와 맨 뒤인 tail을 제어한다.
``` Rust
pub struct Node<T> {
    data: T,
    next_node: Option<NonNull<Node<T>>>,
    prev_node: Option<NonNull<Node<T>>>,
}
```
실제로 data를 저장하는 Node는 위와 같이 구성된다. 각 Node는 실제 데이터 T를 소유하며,  자신의 다음인 next_node, 자신의 이전인 prev_node와 pointer의 형태로 연결된다.

나는 이를 구현하기 위해서 Unsafe Rust를 사용하고야 말았다. 어째서 나는 safe하게 구현하지 못했을까?
# Safety of Rust
Rust는 대부분의 문제를 정적 분석을 통해서 방지하는 것을 모토로 하는 언어다. Rust는 다음의 문제를 **컴파일 타임**에 감지한다.
1) **Dangling Pointer**: 해제된 메모리를 참조
2) **Use-After-Free**: 소유자가 해제한 이후 접근
3) **Double Free**: 같은 메모리를 두 번 해제
4) **Buffer Overflow / Underrun**: 범위를 벗어난 메모리 접근
5) **Data Race**: 동시에 같은 메모리에 읽기+쓰기

Rust는 이러한 문제를 해결하기 위하여 다음의 개념을 도입하였다.
## Ownership
소유권은 대략 아래와 같은 개념이다.
> **소유권은 어떤 객체의 책임을 지는 단 하나의 소유주가 반드시 존재하며, 이 소유권은 객체의 수명과 함께한다.**

즉, 소유권은 어떤 객체가 반드시 소속된 필드가 있어야 함을 의미한다. 
``` Rust
// MyStruct의 운명을 i와 u는 함께한다.
// MyStruct가 i, u를 소유한다.
struct MyStruct {
	i : Type1,
	u : Type2,
};

// 소유권의 이동
// MyStruct는 아래 함수에서 소멸하고,
// MyStruct가 소유한 i의 소유권이 바깥으로 move함.
fn consume_my_and_get_i(my : MyStruct) -> Type1 {
	return my.i;	
	// my의 소멸자 호출됨.
}

// 필드 i를 복사
// Type1이 clonable하면 가능 
fn copy_i_from_my(my : &MyStruct) -> Type1 {
	return my.i.clone();	
	// my의 소멸자 호출됨.
}

fn main() {
	let my : MyStruct = MyStruct::new();

	let t = consume_my_and_get_i(my);

	my.do_something(); // -> fuck !
}

```
쉽게 말하자면, C나 C++처럼 Raw pointer를 여러 객체가 동시에 드는 경우를 금지하는 것.

그래서 Rust는 기본적으로 pointer가 금지입니다. 획득은 되지만, 역참조가 금지임.

포인터 획득이 합법인 이유는 역참조만 안하면, 그냥 값이라서임. 
## Lifetime
꽤 많은 프로그래밍 언어에서 객체의 생애 주기는 흔한 개념이다. 그러나, rust에서는 한 발 더 나아가 다음을 보장해야 한다.

> **참조는 항상 유효하다.**

Rust는 이 lifetime 개념을 확장시켜서 여러 문제를 해결하는데, 그 대표적인 것이 바로 Dangling pointer이다. 기존의 C/C++에서 다음과 같은 행동은 Dangling pointer를 생성하지만, 완벽하게 합법이며, 정상적으로 컴파일된다.
``` C++
Data *make_dangle_ptr() {
	Data d;
	// do something...
	return &d;
}

int main()
{
	// ...
	Data *dangle = make_dangle_ptr();

	// 참조가 out-live 했다.

	dangle->do_something(); // seg fault...
}
```
여기서 `dangle`에 대한 접근이 segfault인 이유는 dangle이 가리키는 실제 값이 더 이상 존재하지 않기 때문이다. 그렇다면, Rust에서는 어떨까?
``` Rust
// 객체 lifetime 예제
fn main(){
	let d : Data = Data::new();
	// do something...

	drop(d); // 명시적인 소멸자 호출

	d.do_something();
}
```
Rust는 lifetime을 바탕으로 객체 d의 생존 시한이 drop 호출에서 종료됨을 계산한다. 이후 d를 인자로 하는 do_something호출에서 lifetime에 어긋나는 호출이 있음을 감지하고, 컴파일을 거부한다.

Rust 컴파일러는 이 lifetime을 내부적으로 정확하게 계산하는데, 이를 위해 lifetime 변수의 개념을 도입하였다.
``` Rust
// example from : https://doc.rust-kr.org/ch10-03-lifetime-syntax.html

// x,y 중 더 긴 것의 참조를 반환하는 함수
// 컴파일 타임에 둘 중 어느 것이 반환될지 모름
// 따라서, 둘 중 수명이 긴 것의 lifetime으로 반환 ref의 수명을 통일함.
// &str은 const char *임. std::string이 아님.
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
	if x.len() > y.len() {
		 x 
	} else { 
		y 
	} 
}
```
위 함수는 두 문자열 a와 b를 받아 둘 중 긴 것을 반환하는 함수이다. str은 문자열의 ref인데, 러스트 컴파일러는 a, b 중 어느 것의 ref가 반환 될 것인지 compile time에 알 수 없다. 따라서, 어떠한 참조의 수명을 의미하는 변수 'a를 도입하여 수명을 표현한다.

> **Rust는 객체의 수명을 표현하는 변수를 사용하여 객체의 유효성을 검사한다.**

위 예제의 경우에는 x,y 중 더 짧은 것의 수명을 `'a`로 하여 반환 된 참조의 수명으로 결정한다.  이를 통해서 다음과 같은 코드가 문제가 됨을 검사할 수 있다.
``` Rust
fn main() { 
	let string1 = String::from("long string is long"); 
	let result; { 
		let string2 = String::from("xyz"); 
		result = longest(string1.as_str(), string2.as_str()); 
		// string2의 수명 종료
	} 
	// result??? -> lifetime error
	println!("The longest string is {}", result); 
}
```
위 예제에서 result는 string1 혹은 string2의 참조인데, result가 string2의 수명이 종료된 이후에 사용됨을 확인할 수 있다. 컴파일러는 lifetime변수 `'a`의 존재를 통해서 lifetime을 검증하고, 문제를 조기에 발견할 수 있다.
## Borrow
대여란 소유권의 이동이 아닌, 소유권의 대여, 즉 참조를 검사한다. 앞서 설명한 life time을 바탕으로 다음을 컴파일 단계에서 다음을 검사한다.

- **한 시점에서 하나의 가변 참조(&mut)만 허용**
- **여러 개의 불변 참조(&)는 허용하지만, 가변 참조와는 공존 불가**
``` Rust
let mut data : i32 = 44;

// 대여의 두 가지 방법
// 일반 참조
let ref_d0 : &i32 = &data;
let ref_d1 : &i32 = &data;
let ref_d2 : &i32 = &data; // 복수의 참조자 허용

// 가변 참조
let ref_mut_d : &mut i32 = &mut data;
//  let ref_mut_d_another : &mut Data = &mut data; // 가변 참조는 한 번에 하나만 가능
//  가변 참조가 생겼으므로, 기존의 ref_d0 등은 유효하지 않음.
```

이들이 금지인 이유는 수학적 증명에 기반하는 탓에, 그 이유를 설명하긴 어렵다. 다만, 이들의 허용이 곧 데이터 레이스의 근본적인 원인이 된다고 한다.
# What is Unsafe Rust?
먼저, Unsafe Rust가 무엇인지에 대해서 알아보자. **Rust의 핵심 철학은 safety이다**. 무수히 많은 글에서, 또 구현 철학에서 안정성을 강조(강요)한다.

> **Safety는 Rust의 모든 것이다.**

그러나 역설적이게도, Rust는 우리에게 이런 safety를 **포기할 수 있는 선택지를 주는 것처럼** 보이는데, 그것이 바로 **Unsafe Rust**이다. Unsafe Rust는 safety를 지키는 것으로 인해서 성능상의 문제가 발생하거나, 구현이 불가능한 경우, 이를 위해 일시적으로 Rust의 safety feature를 해제할 수 있는데, 이러한 사용 방법을 Unsafe Rust라 한다. 
``` Rust
// unsafe rust를 사용하기 위해서는 다음과 같은 unsafe block이 필요하다.
// unsafe block 내의 구현에 대해서는 정적 분석이 동작하지 않는다.
unsafe {
	// do some dangerous shit...
	// compiler does not save your from your mistake ...
}
```
그렇다면, 우리는 다음과 같은 의문을 가질 수도 있다.

>**모든 코드에 unsafe를 박으면 되는 게 아닌가?**
>**Rust가 C++와 다른게 뭐지? 그냥 MZ C++아닌가?**

즉, unsafe rust의 존재 자체가 언어의 철학에 위배되는게 아닌가 하는 것이다. 이는 일견 옳바른 의문처럼 느껴지며, 나도 그렇게 생각했다. 그러나, 여기서 주의해야 할 점은, unsafe의 의미는 safety를 포기하는 것이 아니라는 것이다. 

> **기존의 Rust에서는 언어가 safety를 책임진다면, Unsafe Rust에서는 구현한 자에게 책임이 발생함을 의미한다.**

즉, Unsafe Rust를 구현한 프로그래머는 자신의 코드가 완벽하게 동작함을 문서화 하고, 또 증명해야 한다.
## Unsafe feature
### Raw Pointer
Rust에서 참조를 수행하기 위해서는 ref를 사용해왔다. ref는 borrow-checker의 추적을 받으며, 수명이 항상 원본 데이터를 넘지 않도록, 즉 dangling하지 않도록 엄격하게 관리된다.

그러나, 이 ref에는 몇 가지 치명적인 문제점이 존재하는데, 그중 하나가 바로 **가변 참조가 한 번에 하나만 존재할 수 있다**는 원칙이다. 이는 linked list의 구현에서 치명적인 장애로 존재한다. 이를 해결하는 여러 방법이 있지만, 가장 효율적인 방법은 raw pointer를 사용하는 것이다.

raw pointer는 ref와 달리, borrow checker의 영향을 받지 않는다. 따라서, 각 Node는 pointer를 통해서 서로를 참조하여 double linked list를 구현할 수 있다.
``` Rust
// Node의 동적 할당 
// 유니크 포인터 Box를 이용하여 heap에 Node를 동적으로 할당함.
// into_raw를 사용하여 Box를 제거하고 Box가 가리키던 pointer를 해제하지 않고 반환
// 포인터를 생성하는 것은 Safe한 행동이다.
fn alloc_node(data: T) -> NonNull<Node<T>> {
	let boxed_new_node = Box::new(Node::<T>::new(data));
	NonNull::new(Box::into_raw(boxed_new_node)).unwrap()
}

// List의 가장 마지막 원소 획득
// List의 가장 마지막 Node를 해제하고, 내부의 데이터 T를 반환한다.
// raw pointer를 읽어서 Box로 복원한 다음 data를 가져온다.
// 포인터의 역참조는 항상 Unsafe하다.
fn pop_last(&mut self) -> Result<T, &str> {
        let result = if let Some(node) = self.head {
            unsafe { Ok((*Box::from_raw(node.as_ptr())).get_data()) }
        } else {
            Err("Error : pos out of range.")
        };
        self.head = None;
        self.tail = None;
        self.len = 0;
        result
    }
```
링크드 리스트의 구현에서 중복 가변 참조의 문제보다 더욱 근본적인 문제가 있는데, 이는 바로 **Node의 소유권**이다. 과연 누가 Node의 소유권을 가져야만 할까?

이 또한 여러 구현이 있겠지만, List가 나머지 Node에 대하여 소유권을 갖도록 하는 것이 직관적이다. 그러면, 이를 double linked list에서는 어떻게 구현해야 할까? List는 static sized type이어야 하므로, 가변 개수 Node를 모두 소유하거나 할 수가 없다.

이를 위한 해결책은 없음이다. Linked list 구조와 Node에 대한 소유권은 서로 상충한다. 그렇기에 raw pointer를 사용한 것이다.
### PhantomData - Virtual Ownership of Rust
팬텀 데이터란, 실제로는 해당 자원을 소유하지 않지만, 마치 해당 자원을 소유하는 것처럼 컴파일러가 인식하도록 하기 위해서 존재하는 가상의 데이터이다.

``` Rust
pub struct Iter<'a, T> {
    source: Option<NonNull<Node<T>>>,
    len: usize,
    phantom: PhantomData<&'a Node<T>>,
}
```
위는 list에 구현한 `Iter` 구조체다. `Iter`는 list를 순회하며, list의 각 원소에 대한 참조를 순서대로 제공한다. Iterator를 활용하는 패턴은 Rust에서 아주 일반적인 패턴이며, 이는 safe한 행동이다. 이 safe함에는 당연히 borrow checker의 동작이 전제되어 있다. 그러나, 우리는 List의 구현을 위해서 pointer를 사용하였고, 이는 곧 컴파일러가 Node를 추적할 수단이 없음을 의미한다. 

이러한 경우, 우리는 **"compiler에게 Iter가 T타입 데이터를 논리적으로 참조한다"** 는 것을 알리기 위해서 phantom data를 사용한다. 
# Summary
지금까지 여러 자료구조를 구현하는 과정에서 unsafe rust를 활용, 마치 C를 쓰듯이 여러 구현을 수행했다. 그러나, 공부를 하면 할수록 unsafe rust에는 고려해야 할 지점이 매우 많으며, 그 책임이 막중함을 알게 되었다. 

심지어 현재 구현도 상당히 많은 부분(zero sized type, drop checker, ...)에 대한 고려가 이루어지지 않아서 상당한 개선이 필요한 상황이다. 여기서 소개한 내용은 참고 정도로 보고, 제대로 unsafe rust를 사용하기 위해서는 보다 많은 고려가 필요하다.

# Reference
- Introduction to Algorithms, 4th ed.
- example of lifetime syntax : https://doc.rust-kr.org/ch10-03-lifetime-syntax.html
- Unsafe Rust를 위한 가이드, Rustnomicon : https://doc.rust-lang.org/nomicon/intro.html
