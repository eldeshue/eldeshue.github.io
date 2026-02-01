---
Date: 2024-08-13
tags:
  - Algorithm
  - DataStructure
---
# Prerequisite Algorithm
- [[stack]]
- [[queue]]
# Concept
 [[queue|큐]], [[stack|스택]], [[deque|덱]] 등의 여러 선형 자료구조를 대상으로 단조성을 유지하여 수행하는 최적화 기법. 즉, 선형 **자료구조에 대하여 단조성이라는 invariant를 지키는 것**.
## Monotonicity
단조성은 **함수나 수열이 증가하거나 감소하는 성질**을 의미한다. 즉, 한번 증가하기 시작하면 계속 증가하거나, 한번 감소하기 시작하면 계속 감소하는 성질을 말한다. 단조성은 단조 증가(monotonic increasing)와 단조 감소(monotonic decreasing) 두 가지로 나눌 수 있다.

단조성을 가질 수 있는 데이터는 일반적으로 다음의 조건을 만족할 필요가 있다.
- 데이터 사이에 비교가 가능해야 함. -> Ordering
- 임의의 두 데이터를 뽑았을 때, 두 데이터의 순서가 곧 우열 관계로 이어져야 함.
>ex) Monotonic Decreasing
>> a1, a2, ... an 이라 할 때, i < j에 대하여, ai와 aj가 항상 ai <= aj이다.

---
# Implementation
단조성을 해당 자료구조의 invariant로 유지하기 위해서, 새롭게 원소를 추가하는 경우, 단조성을 해치는 원소를 모두 제거한다.
``` cpp
void push_motonic_stack(std::stack<int> &st, int const data)
{
	// monotonic decreasing stack
	while (!st.empty() && data > st.top())
	{
		st.pop();
	}
	st.push(data);
}
```
---
# When?
그렇다면, 언제 단조성을 이용할 수 있을까?
# Analysis
일반적으로 완전 탐색, O(N^2)의 시간 복잡도를 O(N)으로 줄여주는 경우가 많음.
# Example
- 모노톤 스택   : [[3015. 오아시스 재결합]] 
- 모노톤 큐(덱)  : [[11003. 최솟값 찾기]]
# Summary
- Floyd Parser(플로이드 표현식 파싱 알고리즘)도 모노톤 스택의 응용이라 할 수 있다. 특히 연산자 사이의 우선순위를 바탕으로 파싱 트리의 부모-자식 관계가 결정되는 부분이 그러하다.
- 그 강력함에 비례하여 상당한 고급 테크닉.
- 적용하는 상황을 판별하기에 상당한 관찰이 필요하다고 생각한다.