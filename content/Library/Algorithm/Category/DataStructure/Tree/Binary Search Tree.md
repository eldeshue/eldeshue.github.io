---
Date: 2025-09-30
tags:
  - Algorithm
  - CLRS
  - DataStructure
---
## 1. 이진 검색 트리란?

**이진 검색 트리(BST)** 는 데이터 정렬 및 검색에 매우 효율적인 자료구조이다. 이름처럼 '트리' 구조를 가지며, 각 노드는 최대 두 개의 자식 노드를 가진다. BST가 되려면 반드시 다음과 같은 규칙을 따라야 한다.

> **Invariant of BST**
    1. 모든 노드의 **왼쪽 서브트리**에는 해당 노드의 키(key)보다 **작은** 값을 가진 노드들만 존재
    2. 모든 노드의 **오른쪽 서브트리**에는 해당 노드의 키(key)보다 **큰** 값을 가진 노드들만 존재
    3. 왼쪽과 오른쪽 서브트리 역시 각각 이진 검색 트리여야 함
    4. 중복된 키는 허용되지 않음(별도 구현 필요)
        
이러한 규칙으로, 우리는 데이터를 검색, 삭제, 삽입할 때 평균적으로 $O(\log n)$의 시간 복잡도를 가질 수 있다. 데이터를 순회하는데 $O(n)$이 필요하다.

---

## 2. 핵심 기능 구현: Search, Insert, Delete

이진 검색 트리의 핵심 기능들을 살펴보자.

### 2.1 `search`
`search`는 가장 기본이 되는 연산이다. 주어진 `key`를 가진 노드를 찾는 것을 목표로 한다.

- **알고리즘**:
    1. 루트 노드에서 시작
    2. 현재 노드의 `key`와 찾으려는 `key`를 비교 수행.
    3. **Equal** : 탐색 성공. 현재 노드를 반환.
    4. **Less** : 왼쪽 자식으로 이동하여 2번 과정을 반복.
    5. **Greater** : 오른쪽 자식으로 이동하여 2번 과정을 반복.
    6. 이동하려는 자식이 `None`이면 트리에 값이 없다는 뜻이므로 탐색을 종료.
        
``` rust
// 탐색 함수 구현
// subtree의 root는 None이 아니다. None이면 검색할 이유가 없음.
pub fn search(sub_tree: Rc<RefCell<Node<K, V>>>, key: &K) -> Option<Rc<RefCell<Node<K, V>>>> {
    let mut cur_node = Some(sub_tree);

    while let Some(cur_rc) = cur_node {
        // .borrow()로 내부 데이터에 불변으로 접근
        let compare_result = key.cmp(&cur_rc.borrow().key);

        match compare_result {
            Ordering::Equal => return Some(cur_rc), // 찾았을 경우 Rc를 복제하여 반환
            Ordering::Less => cur_node = cur_rc.borrow().left.clone(), // 왼쪽으로 이동
            Ordering::Greater => cur_node = cur_rc.borrow().right.clone(), // 오른쪽으로 이동
        }
    }
    None // 찾지 못했을 경우
}
```

---

### 2.2 `insert`

`insert`는 BST의 invariant를 유지하면서 새로운 노드를 올바른 위치에 추가하는 연산이다.

- **알고리즘**:
    1. `search`와 동일한 방식으로 `key`가 들어갈 위치를 탐색
    2. 만약 같은 `key`를 가진 노드를 찾으면, 값을 새로 업데이트하고 종료
    3. 탐색이 `None`을 만나 실패하면, 바로 그 자리가 새 노드가 추가될 위치입니다.
    4. 탐색 과정에서 마지막으로 거쳐온 노드(`prev_node`)의 자식으로 새 노드를 연결합니다.
        
- **Rust 코드 들여다보기**:
    - 부모 노드(`prev_node`)의 자식 포인터를 수정해야 하므로, `borrow_mut()`를 호출하여 내부 값에 대한 가변 참조를 얻습니다. 이것이 바로 `RefCell`의 핵심 역할입니다.
    - `prev_node.borrow_mut().left = Some(...)` 와 같이 부모와 자식의 연결을 설정합니다.
    - 새 노드의 부모 포인터는 `Rc::downgrade()`를 통해 `Weak` 참조로 만들어 **소유권 순환을 방지**합니다.
    
``` rust
// 삽입 함수 일부 예시
// ... 탐색 루프 후 ...

// 부모 노드에 대한 Weak 포인터 생성
let weak_prev = Rc::downgrade(prev_node.as_ref().unwrap());
node.set_parent(weak_prev); // 새 노드에 부모 정보 설정

if compare_result == Ordering::Less {
    // .borrow_mut()로 부모 노드의 내부를 수정하여 자식과 연결
    prev_node.unwrap().borrow_mut().left = Some(Rc::new(RefCell::new(node)));
} else {
    prev_node.unwrap().borrow_mut().right = Some(Rc::new(RefCell::new(node)));
}
```

---
### 2.3 `delete`

`delete`는 가장 복잡한 연산입니다. 노드를 삭제한 후에도 BST 규칙이 유지되도록 트리 구조를 재조정해야 합니다.

- **알고리즘**: 삭제할 노드(`z`)의 자식 수에 따라 세 가지 경우로 나뉩니다.
    1. **자식이 없을 때 (리프 노드)**: 가장 간단합니다. 그냥 노드를 트리에서 제거합니다.
    2. **자식이 하나일 때**: `z`를 제거하고, 그 자리를 `z`의 하나뿐인 자식으로 채웁니다.
    3. **자식이 둘** :
	    1. transplant**: 상대적으로 조금 복잡하지만, 안전합니다.
			-  삭제할 대상인 z를 즉시 삭제하고, z의 오른쪽 subtree인 x를 z의 부모에 연결
			- z의 왼쪽 서브트리인 y를  x의 가장 작은 원소의 왼쪽에 연결합니다.
			- 이를 통해서 invariant를 지킬 수 있습니다.
	    2. **swap**: 심플하지만, 참조 유효성을 깨트리기에 문제가 되는 경우가 있습니다
	        - 삭제할 대상인 `z`의 **직후 원소(successor)**, 즉 `z`의 오른쪽 서브트리에서 가장 작은 노드(`y`)를 찾습니다.
	        - `y`를 원래 위치에서 `z`의 위치로 옮깁니다. `y`가 `z`의 모든 연결 관계를 이어받습니다(즉, key와 value만 swap해옵니다). 
            
- **Rust 코드 들여다보기**:
    - `delete` 연산은 여러 노드(삭제될 노드, 그 부모, 자식, 직후 원소 등)의 연결을 동시에 변경해야 함.
    - 이러한 복잡한 재연결 작업을 `transplant`라는 헬퍼 함수로 분리하면 코드가 깔끔해집니다.
    - `transplant`와 `delete` 함수 내에서는 `borrow_mut()`가 빈번하게 사용되어 각 노드의 `left`, `right`, `parent` 포인터를 재설정합니다. `Rc`와 `RefCell` 덕분에 이런 복잡한 '포인터 수술'이 메모리 안전성을 지키며 가능해집니다.   
    - 루트 노드가 삭제될 수도 있으므로, `delete` 함수는 `root` 자체를 변경할 수 있도록 `&mut Option<...>` 형태로 인자를 받습니다.
        
``` rust
// 삭제 함수 일부 예시 (Case 3: 자식이 둘일 경우)
// z는 삭제할 노드
let y = Self::minimum(z.borrow().right.as_ref().unwrap().clone()); // 직후 원소 찾기

// ... y와 z의 위치를 교환하는 복잡한 로직 ...
// 이 과정에서 transplant 함수가 여러 번 호출되며,
// 각 노드의 borrow_mut()를 통해 포인터들이 재설정됩니다.
Self::transplant(root, z.clone(), Some(y.clone()));
y.borrow_mut().left = z.borrow().left.clone();
y.borrow().left.as_ref().unwrap().borrow_mut().parent = Rc::downgrade(&y);
```

## 3. Rust로 구현하기: 소유권이라는 장벽

Rust의 가장 큰 특징은 바로 **소유권(Ownership)** 시스템입니다. 메모리를 안전하게 관리해주지만, 트리처럼 복잡한 연결 구조를 만들 때는 몇 가지 어려움에 부딪힙니다.

- **부모-자식 관계**: 부모 노드는 자식 노드를 '소유'해야 함.(소멸 책임이 발생)
- **자식-부모 관계**: 자식 노드는 자신의 부모가 누구인지 '알아야' 함.(소멸 책임 없음)
    
만약 부모가 자식을 소유하고, 자식도 부모를 소유한다면 어떻게 될까요? 서로가 서로를 놓아주지 않는 **소유권 순환(Reference Cycle)**이 발생하여 메모리 누수(Memory Leak)가 발생합니다.

이 문제를 해결하기 위해 Rust, C++는 **스마트 포인터**라는 강력한 도구를 제공합니다.

---

## 4. 스마트 포인터

### `Rc<T>`: 함께 소유하기 (Reference Counting)

`Rc<T>`는 **참조 카운팅(Reference Counting)** 스마트 포인터, C++의 `sared_ptr<T>` 이다. 하나의 데이터를 여러 '소유자'가 공유할 수 있게 해줍니다.

- **동작 원리**: `Rc`는 데이터에 대한 참조(reference)가 몇 개인지를 계속 셉니다. 새로운 참조가 생길 때마다 카운트가 1씩 증가하고, 참조가 사라지면 1씩 감소합니다. 카운트가 **0**이 되면, 더 이상 아무도 데이터를 사용하지 않는다는 뜻이므로 메모리에서 해제됩니다.
    
- **우리 코드에서**:
    - 부모 노드가 `left`, `right` 자식을 가리킬 때 `Rc`를 사용, 부모가 자식을 소유한다.
    - 이를 통해 하나의 노드가 부모뿐만 아니라, 탐색 중인 임시 변수 등 여러 곳에서 참조될 수 있음.
``` rust
// Node.left 와 Node.right 는 자식 노드를 '공동 소유'합니다.
struct Node<K, V> {
    // ...
    left: Option<Rc<RefCell<Node<K, V>>>>,
    right: Option<Rc<RefCell<Node<K, V>>>>,
    // ...
}
```

### `RefCell<T>`: Interior Mutability

`Rc`는 강력하지만 한 가지 제약이 있다. 여러 곳에서 공유하는 데이터는 **수정할 수 없다(immutable)** 는 **Rust의 Invariant가** 바로 그것이다. 하지만 트리에 노드를 추가(`insert`)하거나 삭제(`delete`)하려면 기존 노드의 `left`, `right` 포인터를 변경할 필요가 있다.

이를 위한 예외적 구현이 바로 `RefCell<T>`이다. `RefCell<T>`는 **내부 가변성(Interior Mutability)** 을 제공하는데, 이를 통해서 Rc가 갖는 제약을 우회한다.

- **동작 원리**: `RefCell`은 Rust의 컴파일 타임 대여(borrow) 규칙을 **런타임에 적용** 합니다. `borrow()`를 통해 불변 참조를, `borrow_mut()`를 통해 가변 참조를 얻을 수 있습니다. 만약 대여 규칙을 어기면(예: 가변 참조가 있는데 또 가변 참조를 요청), 컴파일 에러 대신 프로그램이 실행 중에 `panic`을 일으킵니다.
- 이처럼 대여 규칙을 런타임에 체크하는 만큼, 디버깅에 주의해야 한다.

- **우리 코드에서는?**:
    - `Rc<RefCell<Node<K, V>>>` 형태로 `Rc`와 `RefCell`을 함께 사용합니다.
    - `Rc`로 노드를 안전하게 **공유**하고, `RefCell`을 통해 필요할 때 노드의 내용을 **수정**할 수 있습니다.
``` rust
// prev_node는 Rc<RefCell<Node>> 타입입니다.
// .borrow_mut()를 통해 불변 변수인 prev_node의 내부 값을 수정합니다.
prev_node.unwrap().borrow_mut().left = Some(Rc::new(RefCell::new(node)));
```

### `Weak<T>`: 소유권 없이 참조하기

부모가 `Rc`로 자식을 가리키고, 자식이 `Rc`로 부모를 가리키면 소멸자 호출 과정에서 순환 참조가 발생, 무한 루프에 빠지게 됨. 이러한 문제를 **소유권 순환**이라 하고, 이를 해결하기 위해서 Weak를 도입함. 

> `Weak<T>`는 데이터를 가리키지만 **소유권 카운트를 증가시키지 않는** 포인터이다.

- **동작 원리**: `Weak` 포인터는 대상 데이터의 존재를 보장하지 않습니다. 데이터에 접근하려면 `upgrade()` 메서드를 호출해야 합니다. 이때 데이터가 아직 존재한다면 `Some(Rc<T>)`를, 이미 해제되었다면 `None`을 반환합니다
- Rc가 실제 데이터 T를 가리키면서 접근 가능성을 완벽하게 보장한다면, weak는 T를 포함하는 control block을 가리킨다. 이를 통해서 T의 유효성은 Rc에서 관리되지만, 실제 자원의 소멸 여부는 Weak의 개수에 의해서 결정된다.
    
> **우리 코드에서는?**:    
> 	- 자식 노드가 부모를 가리키는 `parent` 필드를 `Weak<RefCell<Node<K, V>>>`로 선언
> 	- `부모 -> 자식` 관계는 강한 참조(`Rc`)로 유지하고, `자식 -> 부모` 관계는 약한 참조(`Weak`)로 만들어 순환을 해결함.

``` rust
// parent 필드는 소유권을 주장하지 않는 Weak 포인터입니다.
struct Node<K, V> {
    // ...
    parent: Weak<RefCell<Node<K, V>>>,
}

// 새로운 노드의 부모를 설정할 때, Rc::downgrade()를 사용해 Weak 포인터를 만듭니다.
let weak_prev = Rc::downgrade(prev_node.as_ref().unwrap());
node.set_parent(weak_prev);
```

---