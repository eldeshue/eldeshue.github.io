---
Date: 2025-08-29
tags:
  - ProgrammingLanguage
---
# Overview
Rust에서 초기화되지 않은 메모리(uninit)를 어떻게 다루는지 알아보자.

또 이 uninit을 다루기 위해서 도입한 MaybeUninit 래퍼의 의미와 그 사용 방법에 대해서 다룬다.
# Uninitialized?
문자 그대로 실제 값을 할당하지 않은 변수를 의미한다. 
``` rust
let x : i32; // uninit
x = 42i32;   // init
let y = x;   // y : init, x : moved, uninit
x = 33i32;   // x : reinit
```
위와 같이 미초기화, 초기화, 재초기화 등이 가능함을 알 수 있다.

즉, 일반적인 경우에서 초기화는 두 가지 경우에서 발생한다. 하나는 변수를 선언만 하는 것이고, 다른 하나는 move를 통해서 값이 이동한 것이다. 
# Problem - Drop
왜 uninit이 문제가 되는가? 그것은 초기화 되지 않은 메모리 공간에 대해서 drop을 호출하는 위험이 있기 때문이다.

예를 들어 C++에서 copy-assign을 구현할 때, 우리는 기존에 보유하고 있는 자원에 대한 반납을 수행한 다음, 새로이 assign되는 데이터로 초기화하였다. 그리고 이는 rust에서도 마찬가지이다. rust에서도 assign이 발생할 때, 기존 데이터에 대한 drop이 호출된다.

그렇다면, 미초기화 변수에 대한 초기화의 경우, drop이 호출되는 문제는 어떻게 해결할까?
## uninit 추적
rust 컴파일러는 정적 분석을 통해서 변수의 초기화 여부를 대부분 추적할 수 있다. 분석을 바탕으로, rust는 let을 통한 최초의 초기화는 drop을하지 않는다는 원칙을 세웠다.

> **Rust는 정적 분석을 통해서 변수의 초기화 여부를 추적, drop의 호출 여부를 파악한다.**

다만, ref를 통한 할당은 **반드시** drop을 동반한다. 그 이유는 ref는 초기화된 변수에만 생성할 수 있는데, 이는 uninit 데이터를 읽는 것을 방지하기 위함이다. 

> **초기화되지 않은 데이터에 대한 참조 획득은 UB이다. 
> 참조를 통한 초기화는 무조건 drop을 수행한다.**

그러나 다음과 같은 조건부 초기화의 경우에 대해서는 정적인 추적이 불가능하다. Rust는 이러한 정적 분석이 불가능한 경우에 대비하여 runtime에 초기화 여부를 추적한다. 그 예시는 다음과 같다.
``` rust
let condition = true;
let x;
if condition {
    x = Box::new(0);        // `x`는 uninit 상태, drop 안함
    println!("{}", x);
}
// compiler는 x의 초기화 여부를 알지 못함.
// 초기화 여부를 확인, 초기화 되었으면 소멸자 호출

```
runtime에 수행하는 uninit 판단은 stack frame에 flag 형태로 초기화 여부를 저장하여 확인된다. 
# Solution - MaybeUninit
Vec은 uninit한 메모리를 소유하며, runtime에 원소를 추가할 때 마다, 값을 초기화하며, 초기화 영역을 확장한다. 이런 uninit한 메모리에 대한 runtime의 순차적인 초기화는 어떻게 구현할까? 

기존에는 포인터를 활용한 물리적 메모리 카피인 write, copy, copy_nonoverlapping을 활용했다. 이들은 물리적인 메모리 복사이므로, 소멸자의 개입이 발생하지 않는다.

그러나, 포인터를 중심으로 하는 low-level api의 사용은 여러 문제로 이어질 수 있다. 이러한 문제를 해결하기 위한 wrapper가 바로 MaybeUninit이다.

``` rust
use std::mem::{self, MaybeUninit};

// variadic sizable한 배열 구현
// 배열은 본디 필수적으로 초기화가 되어야만 함. 하지만, 가변길이라 기존 방식 초기화 불가능.
// 따라서, 배열을 초기화할 때 [a, b, c] 와 같은 문법을 사용할 수 없음
// 사실 safe하게 구현했으면, iterator를 사용하여 값을 채웠을 것...
const SIZE: usize = 10;

// MaybeUninit을 활용하여 배열 x를 순차적으로 초기화
// uninit으로 배열을 초기화, flag를 false로...
// 이후 초기화에서 drop 호출 안하도록 처리
// 즉 초기화와 drop생략을 양립시키는 테크닉
let x = {
    let mut x: [MaybeUninit<Box<u32>>; SIZE] = unsafe {
        MaybeUninit::uninit().assume_init() // 초기화 되었지만, reassign시 drop안함
    };

    // `MaybeUninit`은 범위 밖으로 벗어나도 아무 일도 일어나지 않습니다.
	// flag가 false임.
    // 따라서 `ptr::write` 대신 단순 assign을 수행해도 drop이 없음
    for i in 0..SIZE {
        x[i] = MaybeUninit::new(Box::new(i as u32));
    }

    // 초기화완료. 배열에서 MaybeUninit을 해소함.
	// 이후 transmute한 배열을 건내받은 쪽에서는 x에 소멸자 호출, 초기화 되었으므로
    unsafe { mem::transmute::<_, [Box<u32>; SIZE]>(x) }
};
// x는 [Box<u32>; SIZE], 즉 단순 배열
```
위 예제에서 uninit한 array를 생성(reserve에 해당)한 다음, for loop를 수행하여 순차적으로 초기화를 하고 있다. 여기서는 배열에 uninit 상태가 발생하기에 MaybeUninit을 통해서 uninit상태를 handling한다.

> **메모리를 MaybeUninit으로 초기화하면, re-assign시 drop호출을 생략한다.**

MaybeUninit은 transparent하여 uninit상태를 처리한 다음, transmute하여 MaybeUninit을 해소할 수 있다. 하지만, 이는 특정 경우(위와 같은 배열 포함)에 대해서 가능함을 보장할 뿐, 대단히 위험하다.

대부분의 경우 safe하게 구현된 적절한 구현(assume init 계열)이 존재하므로, 이를 사용해야 한다. 그 예시는 다음과 같다.
## 값 초기화 패턴
단순한 값은 다음과 같이 `assume_init`으로 초기화 할 수 있다.
``` rust
use std::mem::MaybeUninit;

fn main() {
    let x = MaybeUninit::<u32>::new(10);
    // 안전하게 해소
    let initialized: u32 = unsafe { x.assume_init() };
    println!("{}", initialized);
}
```
x는 `MaybeUninit<u32>`타입으로, assume_init으로 `u32`가 된다. 일종의 casting이다.

단순 값이므로 repr문제는 생기지 않는다.
## 배열 초기화 패턴
위 예제에서는 transmute를 통해서 배열을 초기화 했다. 이를 위한 api가 있는데, 바로 `array_assume_init`이다. 그러나, 이는 아직 stable한 rust로 전환되지 않았다.
## Option VS MaybeUninit
사실 Option의 None을 활용해서 초기화 여부를 표현할 수 있다. 어떻게 보면 Option의 사용은 완벽하게 safe하기 때문에, 더 효과적으로 보일 수도 있다. 그렇다면 왜 MaybeUninit을 사용할까?

이는 Option의 구현에서 None을 표현하기 위해서 추가적인 필드를 사용하기 때문이다. 이 None을 위한 필드의 존재로 인해서 메모리 레이아웃이 바뀌게 되는데, 그로 인해서 unsafe한 초기화의 의의가 사라지게 된다.

이러한 차이에서 볼 때, 논리적으로는 같은 내용을 표현하지만, transparency의 차이로 인해서 option은 적절하지 않을 수 있다. 