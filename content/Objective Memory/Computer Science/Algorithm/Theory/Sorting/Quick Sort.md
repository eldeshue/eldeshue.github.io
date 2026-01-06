---
Date: 2025-06-07
tags:
  - Algorithm
---
# Prerequisite Algorithm
[[Divide and Conquer]]
# Description
in-place하며 unstable한 정렬 알고리즘인 quick sort. 일반적으로 가장 빠른 성능을 보이는 경향이 있어서 quick-sort라는 이름을 얻었다.

quick sort 또한 [[Divide and Conquer]]의 한 예시다.
## 1. Divide
퀵 정렬은 분할 과정에서 잘 선택된 pivot을 기준으로 공간을 둘로 분할한다. 이 때 pivot으로 선택된 값과 비교하여 더 작은 값은 앞 쪽으로, 같거나 더 큰 값은 오른 쪽으로 이동한다. 이후 분할 된 각 부분에 대해서 재귀적으로 정렬을 수행한다. 이후 더 이상 분할할 수 없는 상태, 즉 배열의 크기가 1 이하가 되면 분할을 멈춘다.
### 1.1 Deciding Pivot
pivot의 선택은 divide 된 배열의 size를 결정하므로, 성능에 지대한 영향을 미친다. 최악의 경우는 선택한 pivot의 값이 가장 작을 때 발생하는데, 이 경우 한 번의 분할에서 배열의 size가 1 감소하는 데에서 끝난다.

pivot의 선택에는 다음과 같은 방법이 존재한다.
1. Romuto partition : 가장 뒤 혹은 앞의 원소를 pivot으로 삼는 방식. 구현이 쉽다.
2. Random partition : 난수를 이용하여 pivot을 적당히 구한다. 성능 상의 문제로 유사 난수를 사용.
3. Median of n : n개의 값을 고른 후, 중간 값(median)을 pivot으로 삼는다. 보통 3을 사용한다.
### 1.2 Partitioning
일반적으로는 하나의 pivot을 기준으로 좌우로 데이터를 분할하지만, Dijkstra의 분할 알고리즘을 사용하면 2개의 pivot을 기준으로 3등분이 가능하다.

분할 과정에서 pivot을 기준으로 좌우로 나눠지고, 배열의 크기가 1일 때, 이미 분할되어 있으므로, base case가 된다.
## 2. Conquer
분할 과정에서 이미 대소 비교를 끝냈으므로, 별도의 병합 과정이 필요하지 않다.

> [[Merge Sort]]가 conquer 과정에서 정렬이 이루어졌다면, quick sort는 divide 과정에서 정렬이 수행된다.
# Implementation
## Code
https://github.com/Yeongtong42/UnderthRust/blob/main/algorithms/sort/quick_sort/src/binary_quick_sort.rs

``` Rust

/// partition slice with pivot at end by comp
/// romuto partition
fn partition_by<T, F>(slice: &mut [T], comp: &mut F) -> usize
where
    F: FnMut(&T, &T) -> std::cmp::Ordering,
{
    use std::cmp::Ordering as O;
    let len = slice.len();
    let pivot = len - 1;
    let mut cur_left_pos = 0usize; // points at last element of right side

    for i in 0..len {
        if O::Greater != comp(&slice[i], &slice[pivot]) {
            slice.swap(cur_left_pos, i);
            cur_left_pos += 1;
        }
    }
    cur_left_pos - 1
}

fn quick_sort_by_comp<T, F>(slice: &mut [T], comp: &mut F)
where
    F: FnMut(&T, &T) -> std::cmp::Ordering,
{
    if slice.len() <= 1 {
        return;
    }

    // partition
    let pivot_pos = partition_by(slice, comp);

    // recurse two part
    quick_sort_by_comp(&mut slice[0..pivot_pos], comp);
    quick_sort_by_comp(&mut slice[pivot_pos + 1..], comp)
}

```
## About Code
앞서 분석한 바와 마찬가지로, 분할 과정에서 partition을 수행, 부분 정렬을 수행하므로 conquer 과정에서는 별도의 병합 과정을 수행하지 않아도 정렬이 완성된다.
# Analysis

## Time Complexity - O(N^2)
매 divide마다 선택된 pivot이 가장 작은 값(혹은 가장 큰 값)일 때 최악의 경우가 되며, 재귀 호출 깊이가 O(N)이 된다. divide 과정에서 O(N)이 소모되므로, 최악의 경우 도합 O(N^2)의 복잡도를 갖는다.
## Spatial Complexity - O(N)
매 divide마다 선택된 pivot이 가장 작은 값(혹은 가장 큰 값)일 때 최악의 경우가 되며, 재귀 호출 깊이가 O(N)이 된다. 따라서 최악의 경우, 공간은 O(N)의 복잡도를 갖는다.

최선의 경우는 매 선택된 pivot이 중간 값이 되는 것으로, 이 때 가장 이상적으로 2분할이 수행된다. 따라서 최선의 경우 O(log N)의 비용이 든다.
# Summary
quick sort는 최악의 경우 O(N^2)이 되는 치명적인 문제가 있는 알고리즘이지만, in-place함에서 오는 locality 등의 이유에 힘입어 그 실제 성능은 여타 O(N log N)알고리즘을 압도한다. 그럼에도 불구하고 특정 입력에서 찾아오는 성능 저하를 방지하고자 여러 알고리즘과 함께 사용된다.