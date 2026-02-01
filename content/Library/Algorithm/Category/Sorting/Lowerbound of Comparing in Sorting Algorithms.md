---
Date: 2025-06-16
tags:
  - Algorithm
  - Math
---
# Description
비교 기반 정렬의 성능을 분석하기 위해서 정렬 과정에서 수행되는 비교 횟수의 최솟값을 구해본다. 이러한 비교 횟수에 대한 분석은 알고리즘의 성능에 대한 중요한 이해를 제공하는데, 바로 비교 기반 정렬에서 비교연산의 횟수가 시간 복잡도의 가장 dominant한 비용이 되기 때문이다. 따라서, 비교 횟수의 하한선(lower bound)은 곧 비교 기반 정렬 알고리즘의 성능에 대한 상한선(upper bound)을 구성한다. 
# Modeling - Decision Tree
증명을 위해서 비교 기반 정렬 알고리즘을 Decision tree로 모델링 하고, 이 tree의 특성을 바탕으로 비교 횟수의 최솟값을 분석한다.

decision tree란 어떤 알고리즘을 표현하는 방식으로, 수행하는 과정에서 거치는 상태를 node로 삼아 tree 형태로 표현한 것이다. 최초 상태는 root, 종료 상태는 leaf가 된다. 
## Non-Leaf node
모델링 대상이 되는 decision tree는 leaf를 제외한 모든 노드가 2개의 자식을 갖는 full-binary tree이다. 각각의 non-leaf노드는 정렬 과정에서 수행되는 대소비교를 의미하며, 비교 결과에 따라서 left child 혹은 right child로 분기된다.
## Leaf node
leaf 노드는 정렬이 완료된 상태를 의미한다. 이들은 흔히 인덱스의 permutation으로 표현되는데, 이 때 인덱스는 해당 값들의 최초 위치를 의미한다. 
## 증명
decision tree로 표현된 정렬 알고리즘에서 알고리즘의 수행 과정은 root에서 leaf에 이르는 과정으로 표현된다. 이 path에서 거치게 되는 node의 수는 곧 비교 횟수가 되며, 따라서 해당 알고리즘의 worst case, 즉 최대 비교 횟수는 곧 이 decision tree의 가장 긴 path가 되고, 이는 곧 decision tree의 높이 h가 된다.

따라서, decision tree의 높이 h와 leaf node의 수 l 에 대하여 다음의 과정을 거쳐 비교 횟수의 lower-bound를 구한다. 입력으로 주어진 원소의 개수를 n이라 한다.
### 1) 정렬과 permutation의 관계
적절한 정렬 알고리즘이라면, 가능한 모든 permutation을 정렬 결과로 가질 수 있어야 한다. 즉, 정렬된 상태를 의미하는 leaf node의 개수는 가능한 모든 permutation의 수보다 크거나 같아야 한다. 따라서 다음이 성립한다.
```
n! <= l
```
### 2) perfect binary tree와 leaf node의 관계
decision tree의 node가 대소관계를 묻는 특성 상 자식이 항상 2개이다. 따라서 decision tree는 perfect binary tree이며, 높이가 h일 때, 이 tree의 leaf는 2^h개 존재한다. 따라서 다음이 성립한다.
```
l <= 2^h
```
### 3) 결론
1), 2)의 결론에 의해서 다음이 성립한다.
```
n! <= l <= 2^h
n! <= 2^h
```
여기서 양 변에 log를 취하면
```
h >= log(n!)
```

여기서 양의 정수 n에 대하여 n!보다 n^n이 항상 크기 때문에
```
// 1부터 n까지 곱한 것 보다는 n을 n번 곱한 것이 항상 같거나 크다.
1 * 2 * ...* n <= n * n *...*n
```
다음이 성립한다.
```
h >= log(n^n) > log(n!)
h >= n log(n)
```
앞서 h가 decision tree의 높이이며, 이 높이가 곧 최대 비교 횟수를 의미함을 설명했다. 따라서, 비교 기반의 정렬 알고리즘에서 비교 횟수의 복잡도는 `O(n log n)`이다.