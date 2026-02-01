---
Date: 2025-01-09
---
# Description - IO multiplexing

IO multiplexing이란 현재 상호작용 가능한 복수의 대상(fd, 시그널, 등)에 대하여 OS가 notify해주는 기능으로, 이를 통해서 1대 다수의 통신을 가능하게 함.
# epoll_create
커널의 이벤트 큐 오브젝트를 생성하는 함수.
## Signature
``` C
int epoll_create1(int flags);
```
## Parameters

- flags : 커널 큐 오브젝트의 옵션, EPOLL_CLOEXEC가 유일함.
## Return values

 fd(정상) 혹은 -1(에러)
# epoll_ctl
커널 이벤트 큐가 관리 할 이벤트의 종류를 추가하는 함수
## Signature
``` C
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
## Parameters
- epfd : 커널 큐의 fd
- op : 작업 옵션
	- EPOLL_CTL_ADD : 새로운 대상을 큐에 추가
	- EPOLL_CTL_DEL : 새로운 대상을 큐에서 제거
	- EPOLL_CTL_MOD : 대상 수정
- fd : 작업 대상의 핸들, fd
- event : 작업할 이벤트의 구체적인 내용에 대한 구조체, 추후 설명
## Return values
성공시 0, 실패시 -1
# epoll_wait
커널 큐에서 이벤트의 발생을 대기함.
## Signature
``` C
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```
## Parameters
- epfd : 커널 큐의 핸들, fd
- events : 실행 결과를 받아올 epoll_event의 버퍼의 주소
- maxevents : 한 번에 받아올 이벤트의 최댓값. events 버퍼의 크기와 연계해야 함.
- timeout : 최대 대기 시간, 마이크로 초, -1로 전달하면 이벤트를 무한히 대기함.
## Return values
확인한 이벤트의 개수 혹은 -1(에러)
# Event of epoll
epoll 커널 큐가 관리하는 이벤트를 표현하는 서술자(descriptor). 다음과 같은 구조로 이루어짐.

``` C

struct epoll_event
{
	uint32_t      events;  /* Epoll events */
	epoll_data_t  data;    /* User data variable */
};

union epoll_data 
{
	void     *ptr;
	int       fd; // 해당 이벤트가 발생한 fd
	uint32_t  u32;
	uint64_t  u64;
};

typedef union epoll_data  epoll_data_t;
```
events가 해당 이벤트의 종류를 의미하는 bit mask이다. events에는 여러 종류의 이벤트가 복합되어 들어올 수 있으므로, 이들을 적절한 우선순위에 따라서 순차적으로 처리하는 과정이 필요하다.

해당 구조체는 **epoll_ctl의 인자로 전달하여 이벤트의 등록**에 사용할 수 있으며, 반대로 **epoll_wait의 결과 값으로 받아서 발생한 이벤트를 확인**하는 용도로 쓸 수도 있다.

가능한 이벤트 flag는 다음과 같다.

| 이벤트 플래그            | 설명                     | 등록 필요 여부  |
| ------------------ | ---------------------- | --------- |
| **`EPOLLERR`**     | 소켓 오류                  | **자동 반환** |
| **`EPOLLHUP`**     | 소켓 연결 해제               | **자동 반환** |
| **`EPOLLIN`**      | 읽기 가능한 데이터 도착          | **등록 필요** |
| **`EPOLLOUT`**     | 쓰기 가능한 상태              | **등록 필요** |
| **`EPOLLRDHUP`**   | 반쪽 닫힘 (TCP half-close) | **등록 권장** |
| **`EPOLLPRI`**     | 긴급 데이터 도착              | **등록 필요** |
| **`EPOLLET`**      | Edge-Triggered 모드      | **등록 필요** |
| **`EPOLLONESHOT`** | 이벤트 발생 후 비활성화 (재등록 필요) | **등록 필요** |
## Level-triggered Event vs Edge-triggered Event
상태 그 자체를 기반으로 이벤트를 notify하는 level trigger 방식과 달리, edge triggere 방식은 상태 값의 변화에 따라 notify를 수행한다.

어떠한 소켓의 fd에 대한 read를  edge-trigger하게 감지한다고 하자. 
system buffer가 비어있다가 통신 프로토콜의 동작에 의해서 시스템 버퍼에 read할 데이터가 생기면 이 때 notify를 수행한다.

여기서 level-trigger 방식과 다른 점은 notify의 횟수에 있다. level-trigger 방식은 system buffer에 read할 contents의 유무로 이벤트를 발동하기에 데이터가 남아있기만 하다면, 계속해서 event를 발동한다. 그러나 edge-trigger한 경우, system buffer에 해당 데이터가 도착한 그 순간에만 notify를 수행하고, 이후에는 침묵한다. 

**기본적으로 level-trigger상태이며, epoll_ctl을 호출할 때, EPOLLET옵션을 or해주면 edge-trigger로 이벤트를 설정할 수 있음.**

# Reference
- https://man7.org/linux/man-pages/man7/epoll.7.html