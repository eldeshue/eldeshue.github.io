---
Date: 2025-07-04
tags:
  - Algorithm
---
# Prerequisite Algorithm
- [[Divide and Conquer]]
# Description
주어진 배열에서 임의번째로 작은 원소를 빠르게 찾는 알고리즘.
## Why do we use Selection?
naive하게 생각하면, 주어진 배열을 **정렬**하는 것으로 문제를 쉽게 해결할 수 있다.
-> 뭔 짓거리를 해도 O(N log N)이 들죠?
-> 하지만, 굳이 정렬을 해야만 할까? 더 빠를 순 없을까?

정렬을 하지 않고, 더 적은 비용으로 n번째 원소를 찾아보자.
# Implementation
## Code

``` Rust
/// 슬라이스에서 n번째로 작은 값을 n번째 인덱스로 이동시킵니다.
///
/// 이 알고리즘은 Quick Select를 기반으로 하며,
/// 피벗은 무작위로 선택됩니다. 따라서 평균적으로 O(N) 시간복잡도를 가지지만,
/// 최악의 경우 O(N²)까지 갈 수 있습니다.
/// 후술할 strict 버젼보다 대부분의 경우에 더 빠릅니다.
///
/// # 매개변수
/// - `slice`: 정렬 대상 슬라이스입니다.
/// - `n`: 정렬하여 얻고자 하는 인덱스입니다.
///
/// # 제약
/// - `n >= slice.len()`인 경우 아무 동작도 하지 않습니다.
///
/// # 예시
/// ```
/// let mut slice = [3, 2, 10, 5, -6, 77];
/// select::select_nth_elem_random(&mut slice, 3);
/// assert_eq!(slice[3], 5); // 세 번째로 작은 원소
/// ```
pub fn select_nth_elem_random<T: Ord>(slice: &mut [T], n: usize) {
    let len = slice.len();
    if len == 0 || len <= n {
        return;
    }

    // hoare's partition
    // slice를 pivot 인덱스의 값을 기준으로 좌우로 분할합니다.
    // 작거나 같은 값은 왼쪽으로, 큰 값은 오른 쪽으로...
    let pivot_idx = hoare_partition(slice, random_range(0..len));

    // pivot is not an answer
    if pivot_idx != n {
        // divide
        let range = if n < pivot_idx {
            0..pivot_idx
        } else {
            pivot_idx + 1..len
        };

        // conquer sub-problem
        let offset = range.start;
        select_nth_elem_random(&mut slice[range], n - offset)
    }
}
```
## About Code
select 알고리즘은 전체 slice를 정렬하는 것이 아닌, 탐색 목표로 하는 i번째 특정 원소를 배열의 i번째 위치로 이동 시키는 것을 목표로 한다. **즉, 직접 값을 찾아서 반환하는 것이 아닌, i번째 원소를 i번째 위치로 부분 정렬하는 것이다.** 이 때 i번째 원소보다 작은 원소는 i의 왼쪽으로, i번째 원소보다 크거나 같은 원소는 i의 오른 쪽으로 이동한다. 

select 알고리즘은 [[Divide and Conquer]]의 일종으로, 다음과 같은 과정을 거친다.
### 1. Base Case
``` Rust
let len = slice.len();
    if len == 0 || len <= n {
        return;
    }
```
더 이상 분할이 불가능한 상태이므로, 이탈한다. 또한 탐색 목표인 n이 slice의 크기보다 크므로 배제한다.
### 2. Divide
``` Rust
 // hoare's partition
    // slice를 pivot 인덱스의 값을 기준으로 좌우로 분할합니다.
    // 작거나 같은 값은 왼쪽으로, 큰 값은 오른 쪽으로...
    let pivot_idx = hoare_partition(slice, random_range(0..len));
```
random_range로 얻어낸 pivot을 기준으로 slice를 partition한다. 분할 후 pivot에 해당하는 원소의 위치가 `pivot_idx`가 된다.  이 `pivot_idx`를 기준으로 문제를 분할한다.
### 3. Conquer
``` Rust
 // pivot is not an answer
    if pivot_idx != n {
        // divide
        let range = if n < pivot_idx {
            0..pivot_idx
        } else {
            pivot_idx + 1..len
        };

        // conquer sub-problem
        let offset = range.start;
        select_nth_elem_random(&mut slice[range], n - offset)
    }
```
 앞서 구한 `pivot_idx`를 기준으로 분할한 sub-problem을 정복한다. 만약 탐색 목표인 n이 정확하게 pivot과 같다면, 문제가 풀린 것이므로 재귀 호출은 생략한다.

별도의 merge 과정은 존재하지 않는다.
# Analysis

## Time Complexity - asymptotically O(N), worst O(N^2)
해당 알고리즘의 시간 복잡도를 분석하기 위해서 Divide와 Conquer과정에 집중한다.
### Divide - hoare's partition
해당 과정은 임의의 pivot을 기준으로 pivot이하의 원소는 왼쪽으로, pivot보다 큰 원소는 오른쪽으로 이동한다.

이를 수행하는 알고리즘에는 Lomuto's partition, 혹은 Hoare's partition이 대표적이며, 현재 구현에서는 Hoare의 알고리즘을 사용했다. 해당 알고리즘의 복잡도는 O(N)이다.
### Conquer - worst O(N^2)
Divide 단계에서 분할 된 slice에 대하여 탐색하고자 하는 대상인 n이 존재하는 slice에 대해서 재귀적으로 문제를 해결한다. 

이 때, 탐색 목표인 n이 존재하는 slice를 취하고, 다른 slice는 탐색 대상에서 배제하게 되는데, 이 배제하게 되는 부분의 크기가 중요하다. 최악의 경우는 배제하는 부분의 크기가 1이 되는 경우로, pivot이 최소 혹은 최대가 된 경우에 해당한다. 

즉, 최악의 경우 재귀 호출 당 줄어드는 탐색 범위가 1이 되고, 이를 반복하면 전체 시간 복잡도는 O(N^2)이 되어버린다. **따라서 이를 해결하기 위해서 pseudo random generator를 도입, pivot을 랜덤으로 취하도록 구현하였다.** 따라서 평균 시간 복잡도는 O(N)이 된다.
## Spatial Complexity  - asymptotically O(N), worst O(N^2) 
입력의 크기와 관련된 그 어떠한 공간도 추가로 할당하지 않으므로, 각 호출은 O(1)의 공간 복잡도를 갖는다. 

다만, 해당 알고리즘이 재귀적인 호출 구조를 가지고, 별도의 재귀 최적화를 받지 못하므로, 최악의 경우 O(N^2)에 비례하는 호출 횟수를 갖는다.

따라서 총 공간 복잡도는 O(N^2)이다.
# Summary
별도의 정렬을 수행하지 않더라도, 적절한 partitioning을 반복하는 것을 통해서 우리는 n번째 원소를 n번째 위치로 이동시킬 수 있음을 알았다.

다만, 이러한 알고리즘의 성능은 적절한 pivot의 선택에 큰 영향을 받으며, pivot으로 최대/최소를 선택하는 경우가 곧 최악으로 이어진다. 최악의 선택을 반복하는 경우 해당 알고리즘의 시간 복잡도는 O(N^2)이 된다.

하지만, pivot을 pseudo-random generator로 선택하므로, 기대 성능은 O(N)이 된다. 
# References
- Introduction to Algorithms 4th ed