---
Date: 2025-07-05
tags:
  - Algorithm
---
# Prerequisite Algorithm
# Concept
모든 자료구조의 기초가 되는 기본적인 자료구조. 배열은 다음의 조건을 반드시 만족해야 함.
## Contiguousity
배열을 정의하는 가장 핵심적인 요소. 배열을 구성하는 원소의 연속적인 배치. 결과적으로 배열은 하나의 메모리 블록으로 구성된다.
## Statically Sized Type
배열을 구성하는 원소의 타입은 compile time에 원소의 크기가 결정되는 타입이어야 함.
# Implementation - Random Access
## Code

``` C++
template<typename T>
T random_access(void *const array, size_t const index)
{
    // zero based index
	return *static_cast<T *>(array + sizeof(T) * index);
}
```
## About Code
앞서 설명한 연속성과 정적 타입 조건을 만족하면, 결과적으로 random access가 가능해진다. 

이는 연속성으로 인해서 포인터에 대한 단순 산술 연산으로 특정 원소의 위치를 알 수 있기 때문이며, 또한 static sized type 조건으로 인해 배열을 순회하지 않아도 어느 위치에 원소가 있을지 계산할 수 있기 때문이다.
# Analysis

## Time Complexity
### 접근 - O(1)
단순 산술 연산으로 pointer를 이동시키고, 역참조를 수행한다.
### 삽입 및 삭제 - O(N)
정적 크기 배열은 삽입 및 삭제가 불가능 하지만, 동적 크기 배열은 삽입 및 삭제가 가능하다. 그러나, 변화한 크기에 맞는 연속적인 메모리 구조가 새로 필요하고, 또 변화 후의 데이터를 복사해야 하므로 O(N) 이상의 비용이 필요하다.
## Spatial Complexity - O(1)
단순히 저장하는 자료구조이므로, 원소의 개수에 비례하게 메모리를 점유함.
# Summary
