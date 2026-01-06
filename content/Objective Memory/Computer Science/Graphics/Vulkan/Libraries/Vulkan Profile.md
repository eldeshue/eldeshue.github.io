---
tags:
  - Vulkan
Date: 2025-10-13
---
# Description
Vulkan api를 구성하는 무수히 많은 extension, features를 보다 간편하게 제공하기 위한 라이브러리.

> **Profile이란 extension의 묶음이다.**

일련의 ext와 feature를 profile이라는 형태로 묶어서 관리하고, instance나 device생성 시 일괄 적용하는 hepler 함수를 제공한다.
# Before Use
이 라이브러리는 Vulkan SDK의 일부로 제공된다. 그러나 **SDK에 가면 관련 source가 존재하지 않는다.**

그 이유는 해당 라이브러리가 필요한 profile의 조합에 따라서 **생성되는 라이브러리**이기 때문이다.

프로필은 여러 확장의 모음으로, SDK가 제공하는 프로필도 있지만, 대부분 어플리케이션을 구성하는 쪽에서 custom한 profile이 필요하기 마련이다. 따라서 vulkan profile 라이브러리는 이러한 custom한 profile을 적용할 수 있도록 profile을 구성하는 json파일을 입력으로 받아서 코드를 생성한다.

> 생성 방법은 [sdk 홈페이지](https://vulkan.lunarg.com/doc/sdk/1.4.328.1/windows/profiles_api_library.html)에 자세히 안내되어 있다.

생성된 라이브러리는 단일 헤더(vulkan_profiles.hpp) 혹은 헤더와 소스(vulkan_profile.h + vulkan_profile.c)의 형태로 제공되며, 프로젝트에 추가해서 사용하면 된다. 

해당 라이브러리는 동적 라이브러리 혹은 정적 라이브러리로 별도로 빌드할 필요가 없는 관계로, 별도의 빌드 스크립트가 제공되지 않는다.
# Basic Usage
기본적으로 해당 application에서 필요로 하는 모든 추가기능(extension, feature, etc)을 모두 모은 profile을 구성한 다음, 해당 profile로 instance와 device를 생성하는 것이다. 다만, profile에서 정의하지 않는 extension을 별도로 추가할 수 있다. 이 때 profile로 instance를 생성하는 경우에는 기존과 다른 함수(vpCreateInstance)를 사용하여야 한다.

또 다른 방법으로는 profile까지만 생성하고, 생성한 profile에서 extension을 추가한 다음, 이 extension을 바탕으로 전통적인 방법(vkCreateInstance)로 instance를 생성할 수 있다.

> **Instance를 생성할 때, extension이 중복으로 전달되면 안된다. 따라서 profile에서 ext를 추출해서 vkCreateInstance로 생성하는 방향이 더 유연할 수 있다.**




