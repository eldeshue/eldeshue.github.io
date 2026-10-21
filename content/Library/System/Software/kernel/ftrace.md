---
Date: 2026-10-08
tags:
---
# Overview
커널 디버깅에서 핵심 지표 중 하나인 ftrace와 이를 위한 trace point에 대해서 정리한다.

커널 코드 분석에서 dfs를 하면 그 깊이가 너무나도 깊다. 따라서 일차적으로는 ftrace 이벤트 호출과 그 주변을 우선해서 먼저 보고, 그 다음에 확장하여야 한다.

> kernel code는 마도서라 자격 없는 자는 쉬이 돌아가지 못한다.
# Contents
## trace point
대략 다음과 같은 형태를 띈다.
``` c
// in kernel_clone() ...
trace_sched_process_fork(current, p);
```
해당 함수 호출을 통해서 ftrace 진행. 


# Summary

# Reference