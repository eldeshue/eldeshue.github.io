---
Date: 2025-06-16
tags:
  - Algorithm
---
# Prerequisite Algorithm
# Description
비교를 필요로 하지 않는 정렬 알고리즘. 데이터의 등장 횟수를 counting하는 것 만으로 정렬이 가능함.

이는 각 데이터의 객관적인 위치를 비교하지 않아도 알 수 있는 경우에 가능한 정렬 알고리즘.
# Implementation
https://github.com/Yeongtong42/UnderthRust/blob/main/algorithms/sort/counting_sort/src/lib.rs
## Code

``` Rust

impl<T> CountingSort for &mut [T]
where
    T: Into<usize> + Copy, // T의 데이터를 위한 key값을 얻기 위한 trait
{
    fn counting_sort(self) {
        if self.len() <= 1 {
            return;
        }

        let mut counter = { // 주어진 배열을 순회하며 cout
            let it = self
                .iter()
                .cloned()
                .map(Into::<usize>::into) // 임의의 값 T를 countable한 usize로 변환
                .map(Result::<usize, Never>::Ok);
            get_accumulated_counter(it).unwrap() // count 결과를 바탕으로 accumulate
        };

        let perm = {
            let it = self
                .iter()
                .cloned()
                .map(Into::<usize>::into)
                .map(Result::<usize, Never>::Ok);
            accumulated_counter2permutation(&mut counter, it, self.len()).unwrap()
        };

        apply_permutation_copy(self, &perm);
    }
}

// 이터레이터 it를 순회하면서 각 데이터의 등장 횟수를 count
// 이후 accumulation을 통해서 해당 데이터의 정렬된 위치(인덱스)를 획득하기 편하게 함.
fn get_accumulated_counter<E, I>(it: I) -> Result<Vec<usize>, E>
where
    I: Iterator<Item = Result<usize, E>>,
{
    let mut counter: Vec<usize> = Vec::new(); // 등장 빈도를 저장하는 배열
    let mut max_key = 0;  // key값의 빈도를 저장하기 위하여 필요한 배열의 크기

    for item in it {
        let key = item?; // result이므로 err일 경우 error를 담아서 return
        if counter.len() <= key { // 공간이 부족할 경우 확장, 2의 거듭제곱으로 확장
            counter.resize(key.checked_add(1).unwrap().next_power_of_two(), 0);
        }
        counter[key] = counter[key].checked_add(1).unwrap(); // counter 증가
        max_key = max_key.max(key);
    }

    for i in 1..=max_key { // accumulate
        counter[i] = counter[i].checked_add(counter[i - 1]).unwrap();
    }

    Ok(counter)
}

// 누적 카운터 배열과 아이템 반복자로부터 안정 정렬을 위한 순열(permutation) 배열을 생성
// 획득한 순열을 참조하여 정렬된 위치로 데이터를 이동
// 
// counter[key] = 현재 key값 이하인 원소의 개수
// 0 base index이므로, counter[key] - 1이 key의 index, 즉 perm[key]이다.
fn accumulated_counter2permutation<E, I>(
    counter: &mut [usize], // 누적 카운터
    it: I, // 원본 배열의 이터레이터
    len: usize, // 원본 배열의 크기
) -> Result<Vec<usize>, E>
where
    I: DoubleEndedIterator<Item = Result<usize, E>> + ExactSizeIterator,
{
    let mut perm = vec![0; len]; // 정렬된 위치를 1대1 매핑하는 배열
    for (idx, item) in it.enumerate().rev() { // 역방향으로 순회, stableness
	    // idx : 최초 item의 위치
	    // item : 실제 정렬되어야 할 데이터의 값
        let key = item?;
        counter[key] -= 1; // 다음 저장 위치로 이동
        perm[idx] = counter[key]; // 정렬 결과 이동할 위치 저장
    }
    Ok(perm)
}

// 주어진 순열에 따라 슬라이스의 요소들을 제자리에서 재배치 (swap 사용).
// 이 함수는 `T`가 `Copy` 트레잇을 구현하지 않은 경우에 사용
// panic safety를 구성하기 위함.
fn apply_permutation<T>(src: &mut [T], perm: &mut [usize]) {
    for i in 0..src.len() {
        while perm[i] != i {// 이동할 필요가 없으면 이동하지 않음
            let j = perm[i]; // j : 정렬된 i의 위치
            src.swap(i, j);  // 기존의 i에 위치한 데이터를 j로 이동 데이터 이동
            perm.swap(i, j); // 자리를 바꿨으므로, 기존의 j에 위치한 데이터가 i에 있음
        }
    }
}
```
# Analysis

## Time Complexity - O(N)
counting, accumulation, applying permutation 모두 O(N). 따라서 최종 시간 복잡도는 O(N).
## Spatial Complexity - O(MAX_KEY)
가능한 key값의 가장 큰 값이 곧 메모리의 크기. 즉, key 공간만큼 메모리를 필요로 함.
가능한 key의 범위가 작다면 모르겠지만, 이 key의 범위가 커지면 과도한 메모리를 소모하게 됨.
극단적으로 1000개를 정렬하는데, 공간을 1GB를 사용할 수도 있다.
# Summary
비교를 하지 않아도 해당 데이터의 객관적인 위치를 가리키는 key를 얻을 수 있는 경우에만 가능한 정렬 알고리즘. 다만, key 공간의 크기가 곧 메모리의 비용이므로, 과도한 메모리를 점유할 수 있다.

제한된 key공간이 보장된 경우에만 현실성이 있음.