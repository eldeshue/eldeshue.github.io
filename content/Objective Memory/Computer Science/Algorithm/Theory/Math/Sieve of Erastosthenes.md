---
Date: 2026-01-04
tags:
  - Algorithm
  - Math
---
# Prerequisite Algorithm
# Description
정수 N이하의 소수를 구하는 알고리즘이다. 

소수란 무엇일까? 소수는 그 자신과 1 외에는 나눠지지 않는 수이다. 따라서, 모든 양의 정수는 소수이거나 아니면 소수의 곱으로 표현되는 합성수이다. 

에라토스테네스의 채는 목표로 하는 N이하의 모든 수에 대해서 합성수를 빠르게 제거하여 소수만을 남기는 알고리즘인데, 이 합성수를 제거하고 소수만 남기는 과정이 마치 걸러내는 행위와 유사하다고 하여서 채(sieve)라는 이름이 붙었다.

동작 방식은 다음과 같다.
1. 정수 N에 대하여 N+1 크기의 boolean 배열 `prime`을 준비하고, 모두 true로 설정한다. 
	- 이 boolean 배열이 소수인지 아닌지 판별하는 테이블이 된다.
2. 0, 1은 소수가 아니므로, false로 표기한다.
3. 이후 소수 후보인 i에 대하여 2부터 sqrt(N)까지 순회하며 다음의 작업을 반복한다.
	- `prime[i]`가 `true`인 경우, i는 소수다. 이후 i의 배수는 합성수이므로, `prime`을 순회하며 i의 배수를 `false`로 설정한다.
	- `prime[i]`가 `false`인 경우, 이미 이전 탐색 과정에서 합성수로 판별된 것이다.

> 3번 과정을 N번이 아니라 sqrt(N) 번 반복하는 이유는 합성수를 구성하는 소인수가 짝을 이루기 때문이다. 
> 
>  sqrt(N)보다 큰 소인수 p가 있어서 N을 나눠보자. N/p는 sqrt(N)보다 작을 수 밖에 없고, 이는 채에서 이미 걸러졌을 것이기 때문이다.
# Implementation
가장 단순한 구현은 다음과 같다.
## Code

``` C++
#include <array>

// start부터 stride만큼 이동하며 배열의 값을 초기화
inline void sparse_initialize(
	std::vector<bool> &sieve, 
	uint32_t start, 
	uint32_t stride, 
	bool val) {
	for (size_t i = start; i < sieve.size(); i += stride) {
		sieve[i] = val;
	}
}

// 배열을 받아서 에라토스테네스의 채를 수행, 소수 판정 테이블을 만든다.
void eratosthenes(std::vector<bool> &sieve) {
	if (sieve.size() < 2) return;
	
	// uint32_t const N = sieve.size() - 1;
	std::fill(sieve.begin(), sieve.end(), true);
	sieve[0] = sieve[1] = false;
	
	for (size_t p = 2; p * p < sieve.size(); ++p) {
		if (sieve[p] == true) {
			// p의 제곱부터 시작하는 이유는, 그 밑의 p의 배수는 이미 걸러졌기 때문임
			// 각 호출은 내부적으로 O(N/P)만큼 반복 수행함. 
			sparse_initialize(sieve, p * p, p, false);
		}
	}
}
```

## About Code

# Analysis

## Time Complexity - O(N log (log N) )
해당 알고리즘에서 시간 복잡도를 결정하는 가장 핵심이 되는 행위는 합성수로 밝혀진 수를 false로 초기화 하는 행위, `sparse_initialize` 부분이다.

각 소수 p에 대하여 크기 N+1인 배열에서 p의 배수를 지우는 횟수는 `N/p`가 된다. 이를 N이하의 모든 P에 대해 수행하면 다음과 같이 표현할 수 있다.

$$\sum_{p \le \sqrt{N}, p \in \text{prime}} \frac{N}{p} = N \times \sum_{p \le \sqrt{N}, p \in \text{prime}} \frac{1}{p}$$
여기서  [제2 메르텐스 정리](https://ko.wikipedia.org/wiki/%EB%A9%94%EB%A5%B4%ED%85%90%EC%8A%A4_%EC%A0%95%EB%A6%AC)를 도입하면, 다음이 성립한다.
$$\sum_{p \le x} \frac{1}{p} = \ln \ln x + B + O ( \frac{1}{\ln x}) $$
따라서, 시간 복잡도는 다음과 같다. sqrt는 0.5 거듭제곱이므로, log에서 상수로 무시될 수 있다.

$$\sum_{p \le N} \frac{N}{p} = N \times \ln \ln N$$
## Spatial Complexity - O(N)
소수 판정 테이블의 크기가 N이므로, 공간 복잡도는 N에 비례한다.
# Summary
이렇듯 채를 사용하면 N이하의 소수를 꽤 빠르고 적절한 메모리로 구할 수 있다.

다만, 한계가 존재한다면 숫자의 범위이다. 단순 1M개의 숫자 정도면 순식간에 구할 수 있지만, 현실에서 사용하는 소수는 수십자리 숫자로, N의 단위가 차원이 다르다. 이런 환경에서 소수를 단순한 채로만 구하는 것은 많은 어려움이 따른다(일단 메모리부터 O(N)이라서 답이 없다).

메모리 문제는 차치하고, 단순 계산 시간을 줄이기 위한 많은 최적화가 수행되었다. 다음 코드는 wheeling이라는 테크닉이 적용되어 성능이 개선된 구현이다.

``` C++
#include <vector>

// solver of finding prime number table
// Sieve of Eratosthenes with Wheeling.
class Sieve {
private:
	Sieve() = delete;
	Sieve(Sieve const&) = delete;
	Sieve& operator=(Sieve const&) = delete;

	std::vector<bool> prime;

	inline void sparse_init(
		size_t start,
		size_t stride,
		bool val) {
		for (size_t i = start; i < prime.size(); i += stride) {
			prime[i] = val;
		}
	}

public:
	// Sieve of Erastosthenes with Wheeling 
	explicit Sieve(size_t N) : prime(N + 1, false) {
		if (N < 2) return;
		if (N >= 2) prime[2] = true;
		if (N >= 3) prime[3] = true;
		if (N < 5) return;

		// 5이하의 두 소수 2,3 check
		prime[2] = prime[3] = true;

		// 5이상의 모든 소수는 6K +- 1의 형태를 가짐.
		// 이들에 대해서만 sieve를 수행
		sparse_init(5, 6, true);
		sparse_init(7, 6, true);
		for (size_t p = 5, p_sq = 25; p_sq <= N;) {
			// 6k +- 1의 꼴을 갖는, 다음 소수 후보를 구하는 gap
			size_t gap = (p + 3) % 6; // 6k - 1이면 +2, 6k + 1이면 +4 해줘야 함

			// if p is prime, erase mult of p
			if (prime[p] == true) {
				// 6k +- 1인 값들만 지워야 함
				// 6의 배수가 아닌 p의 배수는 이미 지워짐
				// 따라서 stride에 6을 곱해서 6배 빠르게 계산하기
				size_t stride = p * 6;
				sparse_init(p_sq, stride, false);

				// 만약 p가 6k + 1의 꼴이었다면 위에서는 6k+1만 지웠음
				// p의 배수 중에서 6k - 1의 꼴도 지워줘야 함
				// stride를 6배 크게 잡았기 때문
				// p * (p + gap) = p^2 + p * gap
				if (p_sq + gap * p <= N)
					sparse_init(p_sq + gap * p, stride, false);
			}
			p += gap;
			p_sq = p * p;	// p squared
		}
	}
	Sieve(Sieve&& other) noexcept : prime(std::move(other.prime)) {}
	Sieve& operator=(Sieve&& other) noexcept {
		if (this != &other) {
			prime = std::move(other.prime);
		}
		return *this;
	}

	bool is_prime(uint32_t n) const {
		return n < prime.size() && prime[n];
	}

	std::vector<bool> const& data() const { return prime; }
};
```

> 위 구현은 **나무위키의 [에라토스테네스의 채](https://namu.wiki/w/%EC%97%90%EB%9D%BC%ED%86%A0%EC%8A%A4%ED%85%8C%EB%84%A4%EC%8A%A4%EC%9D%98%20%EC%B2%B4#s-4) 의 예제 코드**를 재활용하고자 class로 묶어서 capsule화 한 것이다.

위 구현의 핵심 아이디어는 탐색 범위 N을 소수가 갖는 다음 특성을 바탕으로 좁히는 것이다.

>>**소수의 특성 : 3보다 큰 모든 소수는 $6k \pm 1$의 형태로 표현할 수 있다.** 
>$6k \pm 2$는 2의 배수이고,  $6k \pm 3$은 3의 배수이며, $6k$는 둘 다이기 때문이다.

즉, 6이 2와 3의 합성수라는 점을 이용하여 **2의 배수와 3의 배수를 미리 제거**하는 것. 단순히 산술적으로 **3배 빨라진다**.

따라서, 기본적으로 5 이상의 모든 수는 합성수로 초기화하고, 5부터 6k-1인 수, 7부터 6k+1인 수에 대해서 채를 치는 것. 이렇게 채의 대상이 되는 pool을 좁히는 테크닉을 wheeling이라고 한다.

에라토스테네스의 채는 범위 내의 소수를 전부 구하는 알고리즘이며, 특정 정수가 소수인지 판별하는 방법은 더 빠른 테크닉이 다수 존재한다.