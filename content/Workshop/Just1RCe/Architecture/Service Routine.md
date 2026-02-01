---
Date: 2025-01-09
---
# Description 

IO multiplexing을 활용한 single-threaded, event-driven한 서비스 루틴.

IO multiplexing을 가능하게 하는 system call에는 select, poll, epoll(linux), kqueue(BSD)등이 있으며, 해당 프로젝트에서는 epoll을 사용할 것임.
