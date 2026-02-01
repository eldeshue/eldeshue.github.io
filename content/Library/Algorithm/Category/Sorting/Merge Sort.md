---
Date: 2025-06-07
tags:
  - Algorithm
---
# Prerequisite Algorithm
[[Divide and Conquer]]
# Description
out-place하며 stable한 정렬 알고리즘인 Merge Sort.

Merge Sort는 von Neumann이 발명한 것으로 알려져 있다.
Merge Sort는 가장 대표적인 분할-정복 알고리즘의 예시인데, 그 과정은 다음과 같다.
## 1. Divide
정렬의 대상이 되는 배열을 정렬이 매우 쉬워질 때 까지 분할한다.

정렬할 대상이 되는 배열의 크기가 1 이하인 경우는 이미 정렬이 완료되었다 볼 수 있으므로, 충분히 쉬운 상태라 할 수 있다. 따라서 크기가 1인 경우를 base case라 하고, 더 이상의 재귀를 멈춘다. 

이러한 base case의 존재가 분할 정복을 시도하기 위한 큰 힌트가 된다.
## 2. Conquer - Merging
divide 과정에서 분할한 두 배열은 재귀 호출 과정에 의해서 정렬된 상태로 돌아오게 된다. 정렬된 두 배열을 병합하여 하나의 정렬된 배열을 얻는다.

병합 과정은 다음과 같다.
1. 정렬된 두 배열의 가장 앞에 각각 포인터 i와 j를 두고, 앞서 allocate해둔 buffer의 앞을 가리키는 포인터 k를 준비한다.
2. i와 j가 가리키는 값을 비교하고, 보다 작은 것을 k가 가리키는 곳에 복사한다.
3. i 혹은 j를 한 칸 이동하고, k도 이동한다.
4. i와 j 모두가 끝에 도달하지 않으면 2번으로 이동하여 병합을 반복한다.
5. 병합이 완료되었다면 buffer의 값 전체를 원본 배열로 복사한다.

즉, 병합 과정을 거치면서 실제 데이터가 이동하며 정렬이 수행된다.
# Implementation
## Code
https://github.com/Yeongtong42/UnderthRust/blob/main/algorithms/sort/merge_sort/src/lib.rs

``` Rust
// implementation of non-recursive merge sort
pub fn merge_sort_by<T, F>(slice: &mut [T], mut comp: F)
where
    F: FnMut(&T, &T) -> std::cmp::Ordering,
{
    // slice size check
    let len = slice.len();
    if len <= 1 {
        // already sorted
        return;
    }

    // buffer allocation
    let merge_buffer;
    let layout = Layout::array::<T>(len).unwrap();
    unsafe {
        merge_buffer = alloc(layout) as *mut T;
    }
    if merge_buffer.is_null() {
        // allocation failed
        panic!();
    }

    // merge sort, non-recursive
	// bottom up
    let mut seg_size = 1;
    while seg_size < len {
        let mut merge_start_pos = 0usize;
        // sort each seg
        loop {
            let begin = merge_start_pos;
            let mid = begin + seg_size;
            let end = std::cmp::min(mid + seg_size, len);
            if mid >= len {
                // already sorted
                break;
            }

            // merge two seg
            let mut l = begin;
            let mut r = mid;
            // merge left and right to the cache
            for i in begin..end {
                unsafe {
                    let next_val =
                        match r == end || (l != mid && (comp(&slice[l], &slice[r]).is_le())) {
                            true => {
                                let tmp = (&slice[l] as *const T).read();
                                l += 1;
                                tmp
                            }
                            false => {
                                let tmp = (&slice[r] as *const T).read();
                                r += 1;
                                tmp
                            }
                        };
                    write(merge_buffer.add(i), next_val);
                }
            }
            merge_start_pos += seg_size << 1;
        }

        // write back ordered seg from cache
        unsafe {
            copy_nonoverlapping(
                merge_buffer,
                &mut slice[0] as *mut T,
                merge_start_pos.min(len),
            );
        }
        seg_size = seg_size << 1;
    }

    unsafe {
        dealloc(merge_buffer as *mut u8, layout);
    }
}
```
## About Code
병합 정렬의 분할-정복에 입각한 분석에 의하면, 이 알고리즘은 분할 과정에서 실재로 아무런 동작도 수행하지 않는다. 따라서 merging과정만 수행하면 되므로, 반복문으로 구현할 수 있다.
# Analysis

## Time Complexity - O(N log N)
Merge Sort는 divide & conquer의 방식을 따르는 전형적인 예시이다. 
### 1. Merging - O(N) 
정렬 과정의 비용은 재귀 함수가 반환되면서 merge 되는 과정에서 발생하는데, 모든 데이터는 각 재귀 깊이마다 한 번 씩 additional buffer로 복사되고, 다시 original buffer로 복사되는 과정을 거친다. 
### 2. Depth of Recursion - O(log N)
또한, 이러한 재귀 함수의 깊이는 정렬하고자 하는 segment의 크기가 1이 될 때 까지 재귀적으로 2분할되므로, 최대 O(log N)이다. 

따라서, 매 depth마다 O(N)의 병합이 발생하기에 전체 비용은 O(N log N)이 된다.
## Spatial Complexity - O(N)
정렬된 두 공간을 병합하는 과정에서 별도의 공간이 필요한데, 그 크기가 최대 전체 size만큼 필요하게 되므로, 그 additional buffer의 크기는 O(N)이다. 
# Summary
병합 정렬은 분할-정복 방법론에서 정복 과정에 집중한 알고리즘으로, stable함으로 인해서 많은 정렬 알고리즘의 근간이 된다.
# Reference
- Introduction to Algorithms 4th ed.