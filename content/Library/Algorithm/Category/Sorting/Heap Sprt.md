---
Date: 2025-05-23
tags:
  - Algorithm
---
# Description
배열을 가지고 표현한 complete binary tree. 부모와 자식 사이에 위계가 성립한다.

이하는 모두 Max Heap을 기준으로 설명한다.
## 배열이란?
인덱스로 접근 가능하며, consecutive한 레이아웃을 갖는, 가장 일반적인 자료구조.
사실 배열이 아니어도 되기는 하는데, 배열이 아니면 의미가 없음.

heap은 배열의 각 칸에 tree의 각 node를 저장한다.
## Complete Tree
모든 노드에 대하여, 오른 자식이 있다면, 왼 자식도 있는 트리. 배열의 앞에서부터 값을 채우고, 이들을 인덱스를 활용하여 tree처럼 사용하는 것이므로, 자연스레 complete tree가 된다. 
## Binary Tree
자식이 둘 있는 tree구조. heap은 배열의 index를 이용하여 노드를 식별하며, 인덱스 사이의 연산을 통해서 부모 및 두 자식 노드를 대단히 빠르게 얻을 수 있음. 즉 node의 id는 해당 node가 저장된 공간의 index임.

일반적으로 index 값 1을 root 노드의 id로 사용하며, shift 연산 및 page size의 이득을 위해서 binary tree로 구성함.
## Invariant of MaxHeap
heap의 invariant는 **"부모가  두 자식보다 큰 값을 갖는다."** 이다. 이러한 invariant가 tree가 갖는 재귀적 성질로 두 자식을 root로 하는 각각의 subtree에 대해서도 적용되면서 heap의 root에 가장 큰 값이 남는다.
## Algorithms
### get_parent
``` rust
fn get_parent(node : i32) -> i32 {
	node >> 1
}
```
### get_children
``` rust
fn get_left_child(node : i32) -> i32 {
	node << 1
}

fn get_right_child(node : i32) -> i32 {
	(node << 1) + 1
}
```
### heapify
heap의 invariant를 유지하는 알고리즘,  heap의 모든 기능은 이 알고리즘에 기반하여 동작한다.
``` cpp
void heapify(std::vector<int> heap, int const cur_node) {
	int const left = get_left_child(cur_node);
	int const right = get_right_child(right);
	if (left < heap.size() && 
		heap[left] >= heap[right] && 
		heap[cur_node] < heap[left])
	{
		// to left subtree
		std::swap(heap[cur_node], heap[left]);
		return heapify(heap, left);
	}
	else if (right < heap.size()  &&
		heap[right] >= heap[left] && 
		heap[cur_node] < heap[right])
	{
		// to right subtree
		std::swap(heap[cur_node], heap[right]);
		return heapify(heap, right);
	}
	// already heapified
	return;
}
```
# Priority Queue
heap의 대표적인 사용처는 우선순위 큐이다. 우선순위 큐는 현재 가지고 있는 원소 중 가장 큰 원소를 반환하는 자료구조다.
## Core Feature
핵심 기능에 대한 부분은 다음 주소의 rust 예제 코드 참조.
- https://github.com/Yeongtong42/UnderthRust/tree/main/collections/binary_heap/src

- comparator.rs : 값들의 비교를 위한 logic 
- heap_logic.rs : heapify를 비롯한 heap의 핵심 logic
- min_heap.rs : 실제 우선순위 큐의 사용을 위해서 노출할 api
## Performance Analysis
- push : heapify에 의해서 O(log n)
- pop : 마찬가지로 heapify에 의해서 O(log n)
- Construct : 놀랍게도 O(n), 교과서 189p 참조
# Reference
- Intruduction to Algorithms



