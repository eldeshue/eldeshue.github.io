---
Date: 2025-09-08
tags:
  - Vulkan
---
# Overview
파이프라인에 자원을 공급하는 과정을 binding이라 한다. 이러한 binding이 어떻게 수행되는지 알아보자.
# Pipeline
파이프라인이란 gpu에서 실행되는 코드, 즉 shader와 이 shader가 실행되는 과정에서 참조하는 데이터를 공급하는 과정까지 포함되어 구성되는 것이다.

셰이더의 인스턴스라 할 수 있겠다.
# Descriptor
파이프라인이 실행되기 위해서는 참조가 필요하다.

이렇게 참조하는 데이터의 대표적인 예시로는 texture가 있다.

다만, pipeline에서는 자원을 직접 전달하지 않고, 해당 자원을 서술하는 일종의 proxy를 둬서 이 proxy를 통해서 참조하게 되는데, 이를 descriptor라 한다.

> **descriptor는 shader에게 참조할 자원의 위치, 즉 포인터라 할 수 있다.**
> **pipeline을 closure에 비유하자면, descriptor는 capture에 상응한다.**

그리고, 이러한 descriptor는 gpu의 효율을 위해서 descriptor set의 형태로 모아서 전달된다.

# Descriptor Set
desc는 개별적으로 binding 되지 않고, desc의 모음, 즉 descriptor set의 단위로 binding된다.

따라서, 이 desc set은 엄데이트 빈도를 기준으로 나눠져야 하며, 어느 정도로 정규화 된 binding 기준이 존재한다.

일반적으로 set 번호를 0, 1, 2의 3단계로 주는데, 숫자가 커질 수록 업데이트 빈도가 커진다.

0은 파이프라인 공통, 2는 매 프레임 마다 업데이트 등으로 구분된다.

descriptor들은 desc pool이 생성과 소멸을 관리한다. descriptor set은 생성이 빈번한 객체이기에 매 번 생성/소멸을 heap에 수행하지 않고, pool을 두어서 관리한다.

# Layout
이 desc의 전달은 결국 shader의 문제이다. 실제 cpu에서 binding하는 정보가 컴파일 된 shader가 정의한 방식, layout이 맞아야 정상적으로 읽어들일 수 있다.

이는 마치 함수의 signature를 호출 과정과 일치시키는 것과 같다.

> 예를 들어 shader에서 texture를 set 1의 binding 2로 읽어들이기로 정의했는데, 셰이더 호출 과정에서 texture를 set 0의 binding 1번으로 전달하면, 이 shader는 정상적으로 호출되지 않을 것이다.

이처럼 컴파일 된 shader와 전달할 자원의 위치 정보를 맞추기 위해서 layout을 정의한다.

## Pipeline Layout
파이프라인의 상태를 정의하는 PSO의 전체 입력 layout을 Pipeline Layout이라 한다.

이는 여러 desc set layout으로 구성된다.

pso자체의 입력을 정의하기 때문에 pso 생성 시 반드시 layout이 정의되어야 한다.

## Descriptor Set Layout
pipeline layout을 구성하는 각 desc set에 대한 layout이다.

이 desc set layout을 바탕으로 desc set을 구성하여 실제 draw명령 recording시 binding이 필요하다.

이 desc set layout에 자원의 정보(size, pos, etc)가 정의된다.
