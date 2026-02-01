---
Date: 2025-08-24
tags:
  - Algorithm
  - CLRS
  - DataStructure
---
# Definition of Dictionary
Dictionary는 삽입(insert), 삭제(delete), 검색(search)이 가능한 자료구조로, 검색을 빠르게 하는 것을 목표로 한다. 

그리고 hash table은 dictionary를 구현하는 한 방법이다.
# Concept 1 - Direct-addressing
 가장 기본적인 dictionary는 array로 구현할 수 있다. 저장하고자 하는 데이터에 고유한 정수 key값을 부여해서 key-value 쌍을 구성하고, 이 key를 배열의 index로 삼는다. 
 
 실제 데이터를 key 값으로 바로 indexing하기에 direct-addressing이라 하며, 정수 key를 index로 하는 배열을 Direct address table이라 한다. 주소의 탐색이 indexing을 통해 이루어지므로, 그 시간 복잡도는 O(1)이 되어 빠른 검색 속도의 핵심이다. 

> **실제 데이터는 배열에 저장하고, 데이터의 key를 index로 삼는다.**

이 때, key로 접근한 address가 반드시 유효해야 한다. 즉, key로 표현 가능한 범위를 반드시 알아야 하며, 이를 바탕으로 배열의 크기를 결정해야 한다. 
## Problem - Wasting Memory 
이러한 구성의 문제는 **메모리 낭비**다. 여기에는 두 가지 측면이 있다.

> **1. 모든 key에 대응하는 memory allocation 필요.**

가능한 모든 key값에 대해서 값을 저장할 수 있어야 하기 때문에, key의 범위만큼 메모리를 미리 할당해야 한다. 그러나, 현실적으로 어떠한 데이터의 조합은 무수히 많이 가능하며, 그에 따라서 key값의 범위가 매우 크거나 무한대가 될 수 있다. 이는 제한된 메모리를 가지는 현실 컴퓨터에서는 불가능하다. 대표적인 예시로 문자열을 들 수 있다.

> **2. 불연속한 key 분포에 대하여, 실제 사용하지 않는 영역에 대한 메모리 사용.**

앞서 살펴본 문제의 경우 array를 dynamic하게 reallocation하는 것으로 대응할 수 있다. 그러나, 사용하는 key의 분포가 sparse한 경우, reallocation의 결과, 중간의 사용하지 않는 영역이 다수 발생하게 된다. 
# Concept 2 - Hash
해시 값(hash value)은 임의의 정수이며, key 대신 index로 사용된다. 그리고 임의의 key 데이터로부터 해시를 계산하는 함수를 해시 함수(hash function)라 한다.

해시의 중요한 요구 사항 중 하나는 **그 값의 범위가 특정한 범위로 제한되어야 한다**는 것이다. 이러한 범위의 제한을 통해서 과도한 memory allocation 문제를 예방할 수 있다.

> **Hash를 이용하여 적당한 크기의 memory를 사용하며 direct addressing을 사용할 수 있다.**

Hash를 통해서 address를 계산하는 이러한 구조를 hash table이라 한다.
## Problem - Hash Collision
### 발생 원인 - 비둘기 집의 원리
hash function의 경우, 서로 다른 key값이 같은 해시를 갖는 문제가 발생한다. 이를 해시 충돌(hash collision)이라 하며, 그 이유는 **비둘기 집의 원리**로 간단하게 생각할 수 있다.

> N마리의 비둘기와 이들이 살기 위한 M개의 비둘기 집이 있다고 하자. 
> 
> 모든 비둘기가 비둘기 집에 살아야 한다고 할 때, N > M이면, 적어도 한 집에는 2마리 이상의 비둘기가  존재한다.
> 
> >**해시를 비둘기에, 메모리 주소를 비둘기 집에 대응하면 hash-collision이 발생하는 이유를 알 수 있다.**

해시 테이블을 사용하는 이유는 memory를 절약하기 위함이므로, hash-collision은 필연적으로 발생한다. 다만, 이에 대응하기 위한 여러 훌륭한 전략이 있다.
### 해결법
#### 1. Chaining - Array + Linked List
hash table을 일종의 divide & conquer를 위한 filter로 바라보는 관점. 테이블의 각 자리에 단일 값 대신, subset을 저장한다. 보통 subset으로 linked list 혹은 그와 유사한 자료구조를 사용한다.
``` C++
// without chaining
std::vector<T>;

// with chainging
std::vector<std::list<T>>;
```

insert할 때, 같은 hash를 갖는 모든 데이터를 같은 subset에 넣고. search할 때 이 subset를 순회하여 key와 일치하는 값을 찾는다.

- 장점 : 간단한 구현으로 collision 해결 가능. **평균 탐색 성능 여전히 O(1).**
- 단점 : **최악의 경우, 단순 linked list로 변함, 탐색 성능이 O(N).**

이상적인 hash function인,  random oracle을 사용한다고 가정하고, 평균 시간 복잡도를 계산하면 여전히 탐색에 O(1)을 만족한다.

>**이상적인 hash algorithm을 사용하여 최악의 경우를 피할 때, 평균 탐색 성능은 O(1)이다.**

대표적으로 C++의 표준 구현체인 `unordered_map/set`이 chainging 메커니즘을 사용한다. 

> Java나 C# 등의 일부 고도로 추상화 된 언어들은 **linked list 대신 검색 균형 트리를 넣는 treeing 전략**을 취하는 경우도 있음. 
#### 2. Open Addressing
collision이 발생했을 때, 대안이 되는 다른 주소를 찾는 방식을 open addressing이라 한다.
##### Avoid Collision - Double Hashing
open addressing은 다음과 같은 방식으로 동작한다.
``` C++
#include <array>
#include <variant>
#include <optional>

template<typename T>
struct LinearProber
{
	using Key = T;
	std::size_t operator()(Key const& k) const noexcept
	{
		return 1;
	}
};

template<typename T,
	typename Hasher_1_ = std::hash<T>,
	typename Hasher_2_ = LinearProber<T>,
	std::size_t _Size>
class HashTable
{
private:
	using Slot = std::variant<bool, T>;	// is_empty, value
	std::array<Slot, _Size> buckets_;
	int num_elem_;
	Hasher_1_ hash1;
	Hasher_2_ hash2;

	HashTable() : num_elem_(0)
	{
		std::fill(buckets_.begin(), buckets_.end(), true);
	}

	bool is_empty(std::size_t pos)
	{
		if (bool* is_empty_ptr = std::get_if<bool>(&buckets_[pos]))
		{
			return *is_empty_ptr;
		}
		return false;
	}

	std::optional<std::size_t> find_empty_or_erased(T const& key)
	{
		std::size_t base_pos = hash1(key) % _Size;
		std::size_t step_size = hash2(key);
		for (int i = 0; i < _Size; ++i)
		{
			std::size_t pos = (base_pos + i * step_size) % _Size;
			if (std::get_if(&buckets_[pos]))
			{
				// empty or erased found
				return pos;
			}
		}
		return {};
	}

	std::optional<std::size_t> find_target_pos_until_empty(T const& key)
	{
		std::size_t base_pos = hash1(key) % _Size;
		std::size_t step_size = hash2(key);
		for (int i = 0; i < _Size; ++i)
		{
			std::size_t pos = (base_pos + i * step_size) % _Size;
			if (is_empty(pos))
			{
				// empty found
				break;
			}
			else if (buckets_[pos] == key)
			{
				// target found
				return pos;
			}
		}
		return {};
	}

public:
	bool insert(T const& elem)
	{
		// no space to push
		if (num_elem_ == _Size)
			return false;

		// open addressing
		// find empty or erased
		if (std::size_t insert_pos = find_empty_or_erased(elem))
		{
			buckets_[insert_pos] = elem;
			++num_elem_;
		}
		return true;
	}
	bool erase(T const& elem)
	{
		// open addressing
		// find T, until empty
		if (std::size_t erase_pos = find_target_pos_until_empty(elem))
		{
			buckets_[erase_pos] = false;
			--num_elem_;
			return true;
		}
		return false;
	}
	bool search(T const& elem)
	{
		// open addressing
		// find T, until empty
		if (find_target_pos_until_empty(elem))
		{
			return true;
		}
		return false;
	}
};
```
여기서 핵심적인 메카닉은 두 개의 hash 함수를 쓴다는 것과, tomb stone이라 불리는 `DELETED`마커의 활용이다.

이 때, step size를 hash함수로 구하지 않고, 1로 고정할 수 있는데, 이러한 구현을 linear probing이라 한다.

놀랍게도 이론적인 성능은 단순 double hashing이 linear probing보다 좋은데, 현대 메모리 구조의 핵심인 caching으로 인해서 linear probing이 더 빨라지는 경우가 자주 있음. 즉, 대충 만든 구현처럼 보이지만, 실전에 강하다.
## Hash Algorithms
그렇다면, 어떻게 hash function을 구현해야 collision을 최소화 하고, 메모리를 절약할 수 있을까?
### Ideal Hashing - Independent Uniform Hashing
다음의 성질을 만족하는 hash 계산 알고리즘을 Independent Uniform Hashing이라 한다. 

> **1. 독립성(Independce) : 서로 다른 두 key가 충돌할 확률은 1/{슬롯의 수}이다.**
> **2. 균일함(Uniformity) : 실행 결과는 해시 범위 내에 고르게 분포한다.**
> **3. 무작위성(Randomity) : 어떤 key에 대한 hash값은 무작위로 결정한다.** 
> >단, 결정된 이후에는 항상 같은 결과를 낸다.

이 모든 조건을 만족하는 알고리즘은 현실에서는 구현할 수 없는데, 메모리에 저장하지 않고 3번을 만족할 수 없으며, 1번을 모든 경우에 대해서 유지하는 것은 수학적으로 불가능하다. 2번은 해시 함수를 호출하는 key의 순서에 의존하므로, 이 또한 말이 안된다.

따라서 이 알고리즘은 이론적인 이상향이며, 이하의 구현 가능한 알고리즘과 비교하는 기준으로 사용된다. 특히 1번과 2번은 hash의 성능에 대한 기준이 된다.

이러한 이상적인 hash function을 **random oracle**이라고 부르기도 한다.
### Static Hashing
특정 해시 알고리즘이 실행 전에 결정되어 있는 경우. uniformity나 independence를 보장하지 않는다.
#### The Division Method
특정 소수로 나눈 나머지를 hash로 사용하는 방법. 계산 속도는 빠르지만, uniformity나 independency가 좋지는 않다.
``` C++
// Simple example of Division Method
#define BUCKET_SIZE 40009; // bucket size must be prime number

size_t hash_by_division(int key)
{
	return key % BUCKET_SIZE; // 특정 소수로 나눈 나머지를 해시로 사용한다.
}
```

bucket size로 소수를 사용하는 이유는 불규칙성 때문이다. 합성수를 사용하면 합성수의 공약수 근처에서 hash가 집중되는 편향성이 존재한다.

runtime에 소수를 구하는 것은 상당히 비쌀 수 있으므로, 미리 소수 테이블을 저장하고, 이를 bucket의 size로 삼아야 하는 까다로움이 있다.

대표적으로 GCC의 C++구현체에서 해당 방법에 기반한 해시 알고리즘을 사용한다.
#### The Multiplication Method
0과 1사이의 실수를 곱한 다음, 그 소수점 아래 부분을 활용하여 해시를 만드는 방법.

``` C++
// simple example of Multiplication Method

#define BUCKET_SIZE 40000; // bucket size가 소수가 아니어도 상관 없음.
#define A 0.78031; // 마찬가지로 자유롭게 설정할 수 있음.

size_t hash_by_mult(int key)
{
	double const ka = key * A;
	return BUCKET_SIZE * (ka - static_cast<int>(ka));// casting과정에서 버림
}
```
bucket의 크기를 임의로 설정해도 된다. 만약 bucket size를 2의 거듭제곱으로 설정한다면, 곱셈/나눗셈을 bit shift로 대체하여 최적화를 수행할 수 있는데, 이 방법을 특별히 Multiply-shift Method라 한다.

이 MUltiply-shift에 randomness를 결합한 여러 알고리즘이 존재하며, 대단히 실용적이라 여겨진다. 대표적으로 MSVC와 Clang에서 이 방식을 사용한다.
### Random Hashing
미리 여러 해시 알고리즘을 준비한 후(이렇게 준비된 일련의 해시 함수들을 family라 함), runtime에 특정 해시 알고리즘을 랜덤으로 선정하여 해당 알고리즘만 사용하는 방법.

해시 알고리즘을 random으로 결정하는 이유는, key값의 분포나 그 데이터 특성으로 인한 independency, uniformity의 저하를 가져오는 특정 입력을 막아 평균 성능을 보장한다. 

> **매 실행마다 hash 알고리즘을 무작위로 선정, 특정 입력을 통한 저격을 예방한다.**

따라서, 같은 key 데이터를 받았다고 해도, 매 실행마다 실제 저장되는 주소가 달라질 수 있다.

hash function family를 어떻게 구축하느냐에 따라서 그 성능이 결정되는데, 여기에는 다양한 방식이 있지만 성능적인 측면에서 multiply-shift method에 factor를 random으로 결정하여 구성하는 방법이 가장 실용적이라 여겨진다.
### Hashing Vector
벡터(문자열 포함)와 같은 매우 긴 데이터에 대해서 hashing을 하는 경우도 빈번하게 발생한다. 이들의 특징은 길이가 가변한다는 점이다. 이들을 위한 해싱 알고리즘은 크게 두 가지가 존재하는데, 하나는 정수론을 응용한 것이고, 다른 하나는 암호에 기반한 것이다. 

벡터는 기존 원시 데이터 타입에 길이를 붙여서 확장(즉, primitive type은 길이가 1인 문자열이다)한 것으로, 현실에서 사용되는 Murmur hash, Siphash 등은 모두 벡터 및 string에 대한 hashing을 지원한다.
### Cryptographic hashing
암호학적 해시 알고리즘은 가변 길이 문자열을 입력으로 받아서 고정된 길이의 정수를 반환하는, 유사 난수 생성 알고리즘이다. 우리가 여러 인증 목적으로 사용하는 키를 생성하는 알고리즘이 여기에 해당한다. 

대표적인 예가 바로 NIST 표준 결정적 암호 해시 함수인 SHA-256이다. 이 함수는 임의의 입력에 대해서 4byte 정수를 반환한다.

이러한 암호학적 해시 함수는 구현이 상당히 복잡하여, 비교적 느린 편이다. 그럼에도 불구하고 이들을 사용하려고 하는 이유는, 이들의 성능이 random oracle에 거의 근접하기 때문이다.

이러한 문제를 하드웨어 가속으로 해결하기도 한다. 최근 하드웨어는 보안이 중요한 부분이기에 여러 암호 관련 기능이 탑재되기 때문이다. 이런 경우 암호학적 해시 함수를 해시 테이블 구현에 사용하기도 한다.
### Hashing Algorihm in Real World
현실 세계에서 주로 사용되는 해시 알고리즘은 앞서 설명한 random hash와는 거리가 멀다. 그 이유는 거대한 hash function family에서 hash function을 선택하는 것이 상당히 비싼 행동이기 때문이다.

현실에서는 계산 속도가 중요하기 때문에 Uniformity에 집중한 static hash, 그 중에서도 division method와 multiply-shift 메서드를 복합하여 최대한 불규칙하게 bit를 섞는 알고리즘을 사용한다. 대표적으로 MurmurHash, CityHash등이 이에 해당한다. 따라서, 여전히 hash collision을 유도하는 HashDos 공격에 취약하다.

최근에는 보안에 대한 중요성이 높아져서 HashDos에 대한 저항성을 가지는 여러 알고리즘이 등장했는데, 그 대표적인 예가 바로 SipHash 알고리즘이다. SipHash는 Random Generated된 seed를 사용하여 Independency와 randomity를 획득했다.
# Practical Tip
## Tip 1 - Hash Combination
C/C++ 등의 언어를 사용하는 경우, 표준 hash table 구현체인 `std::unordered_map`, `std::unordered_set`은 복합 타입(구조체, tuple, 등)을 위한 hash를 제공하지 않는다. 이런 경우 스스로 custom hash를 구현해야 하는데, 이 때 practical하게 hash를 구현하는 방법이 있으니, 바로 hash combination이다.

hash combination은 복합 타입을 구성하는 각 field에 대하여 각각의 hash를 구한 다음, 이들을 적절하게 섞어서 새로운 해시를 구하는 것이다.

나는 Rotation-XOR 방법을 주로 사용하는데, PS 등의 간단한 목적을 위해서는 충분히 유용하다.  방법은 아주 간단한데, 각 필드의 hash를 적절한 순서로 rotation shift를 수행한 다음, 이들을 모조리 xor하는 것이다.

``` C++
#include <unordered_set>
#include <bit>

struct MyStruct
{
	int a;
	long long b;
	double c;

	// necessary for collision detacting
	bool operator==(const MyStruct&) const = default;
};

// Custom type인 MyStruct의 custom hash 구현
// 각 field의 hash를 조합하여 새로운 hash를 만든다.
template<>
struct std::hash<MyStruct>// template specialization
{
	using Key = MyStruct;
	std::size_t operator()(Key const& k) const noexcept
	{
		std::size_t hash_a = std::hash<int>{}(k.a);
		std::size_t hash_b = std::hash<long long>{}(k.b);
		std::size_t hash_c = std::hash<double>{}(k.c);

        // rotl : circular shift
		return std::rotl(hash_a, 1) ^ std::rotl(hash_b, 2) ^ std::rotl(hash_c, 3);
	}
};

int main()
{
	std::unordered_set<MyStruct> test;
	test.insert({ 4, 22LL, 2.5 });
}
```
Rust를 포함한 대부분의 언어, C++의 boost library에서 hash combination을 제공한다.
## Tip 2 - When to use Hash Table?
가끔 hash table을 사용하여 문제에 접근했지만, 해시 테이블의 성능으로 인해서 문제가 되는 경우가 간혹 존재한다. 해시 테이블의 사용이 문제가 될 수 있는 경우를 미리 알아보자.

hash table의 성능이 O(1)이라는 점을 과신하는 경우가 많다.
### Reallocation
collision은 해시 테이블의 대표적인 약점이다. collision의 정도를 판별하기 위한 기준이 바로 load factor이다. load factor는 다음과 같이 구할 수 있다. 
``` Math
load_factor, a = {number of element, N} / {number of bucket, M}
```
hash table에 값을 추가하다 보면, load factor가 1을 넘는 순간이 온다. 이 경우 bucket size를 확장하고, 기존의 모든 원소를 순회하며 각 원소의 hash를 새롭게 구한다.

> **hash table도 resizing을 수행한다.**

hash table의 resizing 비용은 여타의 자료구조에 비해서 결코 싸지 않다(rehashing을 해야 하므로). 이러한 맥락에서 볼 때, 원소의 개수가 많으면 많을수록, 데이터의 크기가 크면 클수록 reallocation이 발생하지 않는 balance tree구조가 적절하다.
### String/Vector
앞서 설명했지만, vector의 해싱은 O(1)이 아니라 O(length)이다. 이름이나 별명, 짧은 길이의 수열 등 제한적인 길이를 갖는 간단한 데이터를 저장/검색하는 문제에서는 충분히 사용할 수 있는 solution이다. 그러나, 그 데이터의 길이가 매우 길거나, 전문적인 검색이 필요한 경우에는 마찬가지로 문제가 된다. 

>**길이에 대한 제한이 존재하지 않는 경우, hash table의 사용은 성능을 보장하지 않는다.**

문자열/vector의 검색 및 저장에는 많은 연구가 진행되었으며, 이들을 위해서 고안된 전용 알고리즘(trie, kmp, bm, aho-korassick, lcp array, 등)이 다수 존재하며 이들을 먼저 고려해야 마땅하다.
## Tip 3 - Chaining VS Open Addressing
앞서 hash collision을 해결하는 두 방법에 대해서 알아봤다. 그렇다면, 어떤 기준으로 두 방법 중 하나를 골라야 할까? 
### Chaining
chaining 방식은 slot에 원소를 직접 저장하는게 아니라, slot에 저장된 subset(linked list)에 저장한다. 그 결과, chaining은 다음과 같은 특성을 갖는다.

- 테이블은 **동적 자료구조**이며, 자원의 삽입/삭제 과정에서 메모리 할당/해제가 발생한다. 그 결과 필연적으로 **메모리 파편화가 발생**한다. 
- subset의 메모리는 테이블 바깥에 위치하므로, **불연속적**이다. 따라서 cache 효율이 떨어진다.
- 같은 초기 해시를 갖는 원소를 탐색하기 위해서 **subset을 순회**한다.

이러한 점을 볼 때, 다음과 같은 경우 chaining이 적합하다.

- 저장할 데이터의 크기가 큰 경우 경우 : dynamic allocation으로 fit한 메모리 사용.
- 데이터의 크기가 예측하기 힘든 경우 : chaining은 범용성이 높아서 데이터의 크기와 상관 없이 구현 가능.
- 데이터의 수(슬롯의 수)가 예측하기 힘든 경우 : reallocation + re-hashing하여 확장 가능.

저장하려는 데이터가 크거나 예측할 수 없을 때, 하드웨어 자원이 풍부할 때 사용함이 적절하다.

> **대부분의 표준 라이브러리의 해시 테이블 구현체는 chainging이므로, 일반적으로는 이쪽을 사용한다.**
### Open Addressing
opend addressing은 slot에 원소를 직접 저장하며, 빈 slot이 없을 때 대안을 찾는 방식이다. 따라서 다음과 같은 특성을 갖는다.

- 테이블은 정적 자료구조이다. 삽입/삭제 과정에서 메모리의 할당/해제가 발생하지 않는다.
- 테이블은 **연속적**이다. 모든 원소가 **인접하게 저장**된다.
- 같은 초기 해시를 갖는 원소를 탐색하기 위해서 **테이블을 순회**한다.

이러한 점을 볼 때, 다음의 경우 open addressing이 적합하다.

- 저장할 데이터가 명백하게 작은 경우 : 빈 slot이 부담되지 않으면 reallocation보다 싸다.
- 저장할 데이터의 수가 예측이 가는 경우 : 테이블의 resizing이 필요가 없다면...
- dynamic allocation에 제한이 있는 경우 : 하드웨어가 약하거나 robust함이 중요할 때 
- **표준 라이브러리의 사용 불가능한 상황**

전체적으로 임베디드 시스템에서 적합한 구현이다.
