---
Date: 2025-08-22
tags:
  - Vulkan
---
# Overview
Vulkan api의 구조에 대해서 정리한다.

이 글은 [여기](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderInterfaceArchitecture.md#layers)의 내용을 정리한 것이다.
# Loader
vulkan은 기본적으로 여러 layer로 구성된 api이다. 이 layer는 api호출 중간에 실행흐름에 개입하여 추가적인 동작을 수행할 수 있도록 한다. 이 layer의 삽입은 동적으로 이루어지는데, 이 layer의 삽입을 관리하는 존재가 loader이다. 응용프로그램은 이 loader와 직접 상호작용한다.

vulkan의 loader는 디바이스 드라이버와 응용프로그램 사이를 매개하며, 여러 작업을 수행한다. layer를 삽입하여 기능을 추가하거나, api함수의 호출을 관리하여 적절한 하드웨어에게 전달되도록 한다. 

## Loader의 목표
loader는 다음의 세 목표를 갖는다.
1. 사용자 시스템의 드라이버를 보조하여 다른 드라이버와의 간섭이 없도록 한다.
2. 동적으로 활성화 할 수 있는 모듈인 layer의 동작을 보조한다.
3. loading으로 인한 overhead를 최소한으로 유지한다.
# Layer
layer는 선택적으로 추가할 수 있는 모듈이다. 이 layer에는 여러 종류가 있으며, 실행 흐름을 후킹(intercept)하여 동작을 변경/추가하거나, 실행 과정을 검증하기도 하는 등 여러 추가적인 작업을 수행한다.

이들은 기존의 api의 중간에 삽입되는 형태로 작동하는데, 이를 loader가 관리한다.
대표적인 예시가 바로 디버깅에 사용하는 validation layer이다.

layer는 일반적으로 library의 형태로 존재하며, VkInstance가 생성될 때 사용할 layer가 결정되고, 활성화된다. 즉 인스턴스는 이러한 방식으로 활성화된 layer를 포함한다.

이러한 layer의 작용을 통해서 프로파일링, 디버깅, 등의 다양한 작업을 실제 api의 변화 없이 수행할 수 있다.

layer는 동적으로 삽입/삭제되므로, 그 실행 과정에서 발생하는 비용이 너무 크거나, 혹은 이미 그 기능이 더 이상 필요하지 않다고 하면 제거할 수 있다.
# Driver
그래픽 api 최하단에는 결국 하드웨어가 존재하며, driver는 이 하드웨어를 소프트웨어적으로 제어한다. vulkan api는 코드나 데이터를 번역하여 gpu가 이해할 수 있도록 한다.

loader는 시스템을 순회하여 유효한 드라이버를 구현한 접근할 수 있는 그래픽 하드웨어를 탐색한다. 이런 드라이버를 추상화 한 객체가 바로 VkPhysicalDevice이다.
# Instance & Device
모든 vulkan api는 두 종류로 나눌 수 있음. 하나는 instance 관련  api이고, 다른 하나는 device 관련임.
## Instance
instance 관련 api는 첫 인자로 instance 관련 객체 핸들을 인자로 받으며, 시스템의 정보(gpu, 등)와 기능(layer, extensions, etc)를 관리한다.
### Instance Object
인스턴스와 관련된 객체는 다음과 같다.
- `VkInstance` : 응용프로그램의 vulkan관련 컨텍스트
- `VkPhysicalDevice` : 현재 시스템에 연결된 gpu를 의미, 성능제한 등을 확인 가능
- `VkPhysicalDeviceGroup`
- etc
### Instance Functions
모든 instance관련 함수는 첫 인자로 instance object를 받는다.

대부분의 instance 함수는 vulkan loader의 header를 include하여 접근할 수 있다. 하지만, 일부 함수의 경우에는 특정한 객체에 bound되는 함수들도 있는데, 이 경우 `vkGetInstanceProcAddr`을 사용해서 해당 객체에 종속된 api를 쿼리해서 사용해야 함. 마치 member 함수와 같음.
## Device
그래픽 하드웨어의 논리적 추상화 객체인 `VkDevice`를 중심으로 하는 api들. 
### Device Object
`VkPhysicalDevice`에서 생성한 `VkDevice`와 그로부터 생성되는 일부 객체. 대부분 실제 렌더링 및 컴퓨팅 과정에 관련된다.
- `VkDevice` : 논리적 gpu
- `VkQueue` : gpu의 커맨드 프로세서, gpu의 여러 엔진에 대응되는 큐
- `VkCommandBuffer` : 큐에 제출될 명령을 기록할 버퍼, CommandList에 대응
- etc
### Device Functions
마찬가지로 device관련 객체를 첫 인자로 받는 함수들. 대부분의 vulkan api는 이에 해당된다.

마찬가지로 device functions들도 `vkGetDeviceProcAddr`로 쿼리하여 획득할 수 있으며, 쿼리에 사용된 디바이스에 종속된다.

# Dispatch Tables & Call Chains
vulkan api는 호출에 첫 인자로 연관된 객체의 핸들을 필수적으로 요구한다. 이는 해당 api가 영향을 미치는 범위를 해당 객체로 제한한다.

이러한 method-like한 구현이 가능한 이유는 객체의 핸들이 실제로는 어떠한 구조체를 가리키는 포인터이며, 이 구조체는 내부적으로 연관된 함수를 매핑하는 테이블을 가지고 있기 때문이다. 이 테이블이 바로 dispatch table이며, vulkan의 loader가 이 테이블을 관리한다.

이 dispatch table은 instance용 device용으로 나눠서 유지하는데, 각각 `vkCreateInstance`/`vkCreateDevice`의 호출 과정에서 loader에 의해 생성된다. 이 때 해당 함수에 연관된 layer가 있다면 해당 layer의 내용을 엮어서 call chain을 형성하게 된다. 즉, call chain은 해당 api 함수와 그에 연관된 layer의 함수를 엮어서 구성한 것으로, 연계되어 호출된다.

> **Dispatch table은 per object, function table로, 해당 객체에 연관된 함수를 저장한다.**

vulkan api함수의 호출은 다음과 같은 과정으로 이루어진다.
```
// Execution of Call-Chain

// Instance functions
app -> trampoline -> Layer 1 -> ... -> Layer n -> terminator -> driver

// Device functions
app -> trampoline -> Layer 1 -> ... -> Layer n -> driver
```
**call chain의 최초 호출은 trampoline 함수**이다. 이 함수는 첫 인자로 주어지는 핸들을 보고, **dispatch table을 참조하여 실제 호출할 함수를 찾아 호출한다.** call chain은 활성화된 layer를 바탕으로, 순차적으로 구성되며 최종적으로는 driver를 호출한다.

예외적으로 **Instance function의 경우 driver전에 terminator함수가 오는데**, 이 함수는 호출할 device의 driver를 결정한다. 이는 instance function이 device function과 달리, 여러 driver를 호출하기 때문으로 보인다. 해당하는 예로는 시스템의 gpu를 순회하는 함수를 생각해볼 수 있다.

> **Call Chain이란 vulkan api의 함수가 이루는 일련의 연쇄적인 호출 구조로, 활성화된 layer의 함수들로 이루어지며, 최종적으로는 driver를 호출한다.** 

이러한 layer에 의한 호출 구조를 유지하기 위해서, loader는 각 오브젝트마다 활성화된 layer와 extension을 모두 기록하고, 그로 인한 call chain을 구성할 수 있어야 한다.

> **call chain 관점에서 layer의 추가는 call chain에 호출될 함수를 추가하는 것이고, extension은 새로운 call chain을 생성하는 것이다.**
> > 즉, layer는 수직적 확장, extension은 수평적 확장에 해당한다.

이는 windows의 COM과 매우 유사하게 생각된다.



