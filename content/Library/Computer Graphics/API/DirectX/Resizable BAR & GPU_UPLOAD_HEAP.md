---
Date: 2025-08-18
tags:
  - DirectX12
  - DirectX
  - Graphics
---
# Overview
Resizable BAR과 이를 응용한 D3D12의 기술인 GPU_UPLOAD_HEAP에 대해서 다룬다.

해당 내용은 유영천님의 [영상](https://www.youtube.com/watch?v=CpE4TkaMwO8&t=3121s)을 요약 및 정리한 것임을 밝힌다.
# 배경 - GPU가 자원을 소비하는 방식
GPU로 데이터를 전달하는 동작은 기본적으로 GPU가 수행한다고 이해하는 것이 적절함. 사실은 DMA 컨트롤러를 비롯한 별도 하드웨어의 동작일 가능성이 높아보임.
## 1. Asynchronous DMA Transfer - read from vram
1차적으로 메인 메모리에 데이터를 올린 다음, gpu의 vram으로  비동기적 이동, 최종적으로는 vram에서 자원을 사용하도록 하는 방식. 다수의 복사가 발생함.

vram에 적재하는 것이 가장 접근 속도가 빠르기에, 시스템 메모리를 거치는 비용을 지불하고 명시적으로 데이터를 옮기는 것.

다만, 메인 메모리에서 벌어지는 **추가적인 복사(upload heap에 데이터 복사) 및 gpu의 비동기적인 복사 수행(gpu가 언제 다 읽어가는지 확인 필요)** 등으로 인하여 자원 관리가 까다로움.

비동기 업로드 매니져라는 자료구조를 통해서 문제를 해결할 수 있으나, 구현이 복잡함.
## 2. GART - read from main
GPU Address Remapping Table, GPU의 MMU가 main memory를 mapping한다. 메인 메모리에 데이터를 두고, GPU로 하여금 필요에 따라서 메인 메모리에 접근하여 사용하도록 하는 방식. 

CPU와 GPU가 동시에 접근하기에 resource barrier를 통한 접근 통제가 필요하다.

DX12를 기준으로, upload heap에 해당하며, CBV나 UAV, Vertex buffer 용도로는 사용할 수 있다.

하지만, 수 GB에 달하는 대용량 texture 등에는 사용하기 대단히 곤란하다.
# 문제 제기 - 왜 바로 vram에 쓸 수 없는가?

## NUMA 아키텍쳐
기본적으로, 해당 문제는 외장 gpu, 즉 dgpu환경에서만 발생하는 문제다. UMA 아키텍쳐에서는 시스템 메모리와 vram의 구분이 존재하지 않는다. 둘 모두 하나의 address space 상에 존재하며, 따라서 데이터의 이동 또한 자유롭다.

UMA 아키텍쳐를 채용한 사례로는 PlayStation, Xbox, AppleSilicon이 대표적이다.
## VRAM에 대한 접근은 메모리 접근이 아니라 IO다.
CPU가 vram에 바로 write할 수 없는 이유는 vram은 gpu의 메모리이고, 이에 대한 접근은 IO작업이기 때문이다. 따라서 시스템 콜을 통해서 수행되어야 하며, 이는 인터럽트 등 수행 완료를 기다리게 되어 매우 느리다.
## MMIO에는 크기 제한이 존재한다.
이러한 문제를 해결하기 위해서 Memory Mapping, 즉 MMIO를 고려할 수 있다. 그러나 이 또한 제한이 존재하는데, 바로 크기 제한이다. 

MMIO를 위해서는 OS가 장치에 virtual address를 할당할 필요가 있다. 할당된 virtual address에 대한 접근을 주변 장치에 대한 접근으로 변환하기 때문이다. 이 때 필요한 것이 바로 BAR이다.

---
### What is BAR?
BAR는 Base Address Register의 약자로, memory mapping된 하드웨어의 주소를 위한 offset을 저장하는 register를 의미한다.

과거 구형 OS는 MMIO를 위한 주소가 고정된 상태로 존재했다. 그 결과, 새로운 장치의 추가는 address의 겹침으로 이어져 문제가 되었다.

따라서 현대 OS는 하드웨어의 memory를 메인 메모리에 매핑할 때, **매핑되는 주소를 동적으로 결정한다**. 이 BAR에 저장되는 주소값을 바탕으로 virtual address에 매핑된 메모리 영역을 계산한다.

 이러한 동적 매핑은 booting과정에서 bios에 의해 PCIe로 연결된 장치를 탐색하는 과정에서 이루어지며, 연결된 장치 상황에 따라 그 주소 offset이 결정된다. 
 
 ---
기존의 MMIO의 경우, **mapping되는 메모리 영역의 크기가 최대 256MB**였다. 그러나 GPU의 vram은 수 GB를 쉽게 넘었고, 그래픽스에서 사용되는 여러 고해상도 texture의 용량 또한 매우 커져서 256MB로는 대단히 제한적이었다.
# Resizable BAR
이러한 BAR의 크기 제한을 극복하여 vram에 대한 MMIO를 가능하게 한 기술이 바로 Resizable BAR, 줄여서 RBAR이다. RBAR을 구현한 하드웨어에서는 권한의 재협상을 통해서 256MB의 제한을 넘는 크기로 memory mapping 범위를 확장할 수 있다. 

> **RBAR은 수 GB에 달하는 GPU의 vram 전체를 메인 메모리에 mapping하는 것이 가능하게 했다.**

그러나, 이러한 RBAR은 하드웨어(main board, gpu, pcie, 등) support를 받아야 가능한 기술이다. 따라서, 디바이스 컨텍스트 초기화 과정에서 현재 하드웨어가 해당 기능을 지원하는지 여부를 체크하고 사용해야 하며, 지원하지 않는 경우를 위한 실행 흐름도 반드시 준비해야 한다. 
## Performance
RBAR을 통한 MMIO로 gpu 자원을 다루는 것은 대단히 효율적이다. MMIO를 통해서 메인 메모리를 거치지 않고, vram에 직접 데이터를 전달하는 만큼, 가능한 방법 중 가장 빠르다. 

물론 MMIO의 특성 상, 어느 정도의 오버헤드가 존재하기는 하는데, 이를 write combine 방식의 효율적인 캐싱 방법으로 이를 극복했다. 다만, 진짜 vram이 아니기 때문에 여전히 write는 느리다. 
# Reference
- 유영천님의 유투브 영상 : https://www.youtube.com/watch?v=CpE4TkaMwO8&t=3121s