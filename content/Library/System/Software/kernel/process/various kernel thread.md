---
Date: 2026-10-14
tags:
  - Linux
  - Process
---
# Overview
다양한 커널스레드에 대해서 다룬다.
# Contents
## kthreadd
- **pid: 2**, user space로 분화되는 init 다음으로 존재
- handler: kthreadd()

> 커널 스레드를 생성하는 커널 스레드, 궁극의 조상

``` bash
# ps axjf
~> ps axjf | head -20
   PPID     PID    PGID     SID TTY        TPGID STAT   UID   TIME COMMAND
      0       2       0       0 ?             -1 S        0   0:00 [kthreadd]
      2       3       0       0 ?             -1 S        0   0:00  \_ [pool_workqueue_release]
      2       4       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-rcu_gp]
      2       5       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-sync_wq]
      2       6       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-kvfree_rcu_reclaim]
      2       7       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-slub_flushwq]
      2       8       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-netns]
      2      10       0       0 ?             -1 I<       0   0:00  \_ [kworker/0:0H-events_highpri]
      2      13       0       0 ?             -1 I<       0   0:00  \_ [kworker/R-mm_percpu_wq]
      2      15       0       0 ?             -1 S        0   0:02  \_ [ksoftirqd/0]
      2      16       0       0 ?             -1 I        0   0:09  \_ [rcu_preempt]
      2      17       0       0 ?             -1 S        0   0:00  \_ [rcu_exp_par_gp_kthread_worker/0]
      2      18       0       0 ?             -1 S        0   0:00  \_ [rcu_exp_gp_kthread_worker]
      2      19       0       0 ?             -1 S        0   0:00  \_ [migration/0]
      2      20       0       0 ?             -1 S        0   0:00  \_ [idle_inject/0]
      2      21       0       0 ?             -1 S        0   0:00  \_ [cpuhp/0]
      2      22       0       0 ?             -1 S        0   0:00  \_ [cpuhp/1]
      2      23       0       0 ?             -1 S        0   0:00  \_ [idle_inject/1]
      2      24       0       0 ?             -1 S        0   0:00  \_ [migration/1]
```
모든 커널 스레드의 ppid가 2, 즉 kthreadd를 부모로 가지는 것을 확인할 수 있다.
## kworker
- handler: worker_thread()

`process_one_work()` 호출하여 워크 큐를 소비하며 작업을 수행하는데, user space의 thread pool마냥 범용 한 비동기 작업 처리를 수행함

> 워크 큐에 큐잉된 work를 실행하는 프로세스
## ksoftirqd
- handler: run_ksoftirqd()

> **soft IRQ**의 작업을 받아서 처리하는 per cpu한 범용 스레드

softirq는 IRQ 핸들링 중 일부 작업을 지연 처리하는 매커니즘이다. softirq는 IRQ 복귀 과정에서 수행되는데, 작업량이 많은 경우 cpu를 과점하여 user process에 대한 서비스가 소홀해질 수 있다.

이를 해결하기 위해 일정 이상의 작업은 별도의 전용 thread에서 일괄 처리되는데, 이를 수행하는 커널스레드가 ksoftirqd이다. 

ksoftirqd가 수행하는 softirq핸들러는 여전히 IRQ 컨텍스트에서 실행되는 구현이라 scheduling을 고려하지 않으며, ksoftirqd는 이런 여러 softirq 핸들러를 모아서 실행만 시켜준다.

softirq의 cpu 점유는  IRQ 맥락에서 수행되기에 정식 스케줄링을 통한 cpu 점유가 아니므로 선점이 불가능하며, 따라서 코어를 오랫동안 점유할 위험이 있다. 

따라서, softirq는 일반적인 디바이스 드라이버가 호출할 수는 있어도, 내부를 구현할 수 있는 방식이 아니다. 다음과 같이 커널 내부에 미리 정의된 경우만 softirq로 핸들링될 수 있다.

``` c
// include/linux/interrupt.h, line 519
enum {
    HI_SOFTIRQ = 0,
    TIMER_SOFTIRQ,
    NET_TX_SOFTIRQ,
    NET_RX_SOFTIRQ,
    BLOCK_SOFTIRQ,
    IRQ_POLL_SOFTIRQ,
    TASKLET_SOFTIRQ,
    SCHED_SOFTIRQ,
    HRTIMER_SOFTIRQ,
    RCU_SOFTIRQ,

    NR_SOFTIRQS
};
```

> [[NAPI]]에서 다룬 softirq packet 처리의 원리

> ksoftirqd의 경우 코어당 1개 존재
## threaded IRQ
- handler: 다양한 IRQ 핸들러

> **특정 하드웨어 IRQ**의 작업 전체를 받아서 처리하는 전용 스레드

두 종류의 핸들러로 구성되는데, primary handler는 IRQ 발생 직후 호출되어 sleep 중인 스레드를 wakeup하고, 이 스레드가 스케줄링을 거쳐서 실제 IRQ 핸들링을 수행한다.

threaded IRQ는 애초에 **IRQ 핸들링 자체를 스레드 위에서 할 것**을 전재로 작성된다. 따라서 기존 IRQ 핸들링 맥락에서 금기였던 `sleep`, `schedule` 모두 가능하다.

시스템에서는 다음과 같은 형태로 확인이 가능하다

> irq/{irq 번호}-{작업 내용}

``` bash
root          79  0.0  0.0      0     0 ?        S    10월04   0:27 [irq/9-acpi]
root          92  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/120-dmar0-prq]
root         101  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/125-pciehp]
root         102  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/126-pciehp]
root         103  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/127-pciehp]
root         555  0.0  0.0      0     0 ?        S    10월04   0:13 [irq/144-SYNA8004:00]
root        2200  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/173-iosm]
root        2265  0.0  0.0  81460  4500 ?        Ssl  10월04   0:08 /usr/sbin/irqbalance
root        2272  0.0  0.0      0     0 ?        S    10월04   0:07 [irq/174-iwlwifi:default_queue]
root        2273  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/175-iwlwifi:queue_1]
root        2274  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/176-iwlwifi:queue_2]
root        2275  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/177-iwlwifi:queue_3]
root        2276  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/178-iwlwifi:queue_4]
root        2277  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/179-iwlwifi:queue_5]
root        2278  0.0  0.0      0     0 ?        S    10월04   0:01 [irq/180-iwlwifi:queue_6]
root        2279  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/181-iwlwifi:queue_7]
root        2280  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/182-iwlwifi:queue_8]
root        2281  0.0  0.0      0     0 ?        S    10월04   0:00 [irq/183-iwlwifi:exception]
root        2475  0.0  0.0      0     0 ?        S    10월04   0:01 [irq/184-AudioDSP]
root      112140  0.0  0.0      0     0 ?        S    10:09   0:00 [irq/171-mei_me]
```

> android와 같은 모바일 제품에서는 threaded IRQ를 적극 사용한다고 한다, 대략 50개. 

> 보유한 x86-64 페도라의 경우 약 20여개.
# Summary

## IRQ 선택의 기준
![549](data:image/svg+xml;charset=utf-8,%3Csvg%20font-family%3D%22-apple-system-body%2C%20ui-sans-serif%2C%20-apple-system%2C%20system-ui%2C%20%26quot%3BSegoe%20UI%26quot%3B%2C%20Helvetica%2C%20%26quot%3BApple%20Color%20Emoji%26quot%3B%2C%20Arial%2C%20sans-serif%2C%20%26quot%3BSegoe%20UI%20Emoji%26quot%3B%2C%20%26quot%3BSegoe%20UI%20Symbol%26quot%3B%22%20font-weight%3D%22400%22%20data-d-component%3D%22svg%22%20fill%3D%22currentColor%22%20style%3D%22color%3Argb\(237%2C%20237%2C%20237\)%22%20viewBox%3D%220%200%20340%20405%22%20width%3D%22100%25%22%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%3E%3Crect%20x%3D%2290%22%20y%3D%228%22%20width%3D%22160%22%20height%3D%2242%22%20rx%3D%229%22%20fill%3D%22%2364748b%22%20fill-opacity%3D%22.14%22%20stroke%3D%22%2394a3b8%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%2234%22%20font-size%3D%2213%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%20font-weight%3D%22600%22%3EIRQ%20%EC%B2%98%EB%A6%AC%20%EC%84%A4%EA%B3%84%3C%2Ftext%3E%3Cpath%20d%3D%22M170%2050%20V83%22%20stroke%3D%22%2394a3b8%22%20stroke-width%3D%221.6%22%2F%3E%3Cpath%20d%3D%22M170%2083%20L250%20128%20L170%20173%20L90%20128%20Z%22%20fill%3D%22%230d9488%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%230d9488%22%20stroke-width%3D%221.4%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22123%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20fill%3D%22currentColor%22%3ESleep%20%2F%20Blocking%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22138%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20fill%3D%22currentColor%22%3E%ED%95%84%EC%9A%94%ED%95%9C%EA%B0%80%3F%3C%2Ftext%3E%3Cpath%20d%3D%22M250%20128%20H285%20V190%22%20stroke%3D%22%230d9488%22%20stroke-width%3D%221.6%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22270%22%20y%3D%22118%22%20font-size%3D%2211%22%20fill%3D%22%230d9488%22%3EYes%3C%2Ftext%3E%3Crect%20x%3D%22219%22%20y%3D%22190%22%20width%3D%22115%22%20height%3D%2248%22%20rx%3D%228%22%20fill%3D%22%230d9488%22%20fill-opacity%3D%22.13%22%20stroke%3D%22%230d9488%22%2F%3E%3Ctext%20x%3D%22276%22%20y%3D%22209%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%20font-weight%3D%22600%22%3EThreaded%20IRQ%3C%2Ftext%3E%3Ctext%20x%3D%22276%22%20y%3D%22225%22%20font-size%3D%2210%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3Esleep%20%EA%B0%80%EB%8A%A5%ED%95%9C%20%EC%B2%98%EB%A6%AC%3C%2Ftext%3E%3Cpath%20d%3D%22M170%20173%20V207%22%20stroke%3D%22%2394a3b8%22%20stroke-width%3D%221.6%22%2F%3E%3Ctext%20x%3D%22182%22%20y%3D%22187%22%20font-size%3D%2211%22%20fill%3D%22%2364748b%22%3ENo%3C%2Ftext%3E%3Cpath%20d%3D%22M170%20207%20L247%20253%20L170%20299%20L93%20253%20Z%22%20fill%3D%22%2364748b%22%20fill-opacity%3D%22.10%22%20stroke%3D%22%2394a3b8%22%20stroke-width%3D%221.4%22%2F%3E%3Ctext%20x%3D%22170%22%20y%3D%22248%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%EC%A7%A7%EA%B3%A0%20%EC%8B%A4%ED%96%89%20%EC%8B%9C%EA%B0%84%EC%9D%B4%3C%2Ftext%3E%3Ctext%20x%3D%22170%22%20y%3D%22263%22%20font-size%3D%2212%22%20text-anchor%3D%22middle%22%20fill%3D%22currentColor%22%3E%EC%98%88%EC%B8%A1%20%EA%B0%80%EB%8A%A5%ED%95%9C%EA%B0%80%3F%3C%2Ftext%3E%3Cpath%20d%3D%22M93%20253%20H54%20V333%22%20fill%3D%22none%22%20stroke%3D%22%2394a3b8%22%20stroke-width%3D%221.6%22%2F%3E%3Ctext%20x%3D%2261%22%20y%3D%22242%22%20font-size%3D%2211%22%20fill%3D%22%2364748b%22%3EYes%3C%2Ftext%3E%3Crect%20x%3D%227%22%20y%3D%22333%22%20width%3D%22112%22%20height%3D%2251%22%20rx%3D%228%22%20fill%3D%22%23d97706%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%23d97706%22%2F%3E%3Ctext%20x%3D%2263%22%20y%3D%22353%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20fill%3D%22currentColor%22%20font-weight%3D%22600%22%3EHard%20IRQ%3C%2Ftext%3E%3Ctext%20x%3D%2263%22%20y%3D%22370%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3E%EC%A6%89%EC%8B%9C%20%EC%B2%98%EB%A6%AC%3C%2Ftext%3E%3Cpath%20d%3D%22M247%20253%20H289%20V333%22%20stroke%3D%22%2364748b%22%20stroke-width%3D%221.6%22%20fill%3D%22none%22%2F%3E%3Ctext%20x%3D%22263%22%20y%3D%22242%22%20font-size%3D%2211%22%20fill%3D%22%2364748b%22%3ENo%3C%2Ftext%3E%3Crect%20x%3D%22220%22%20y%3D%22333%22%20width%3D%22114%22%20height%3D%2251%22%20rx%3D%228%22%20fill%3D%22%232563eb%22%20fill-opacity%3D%22.12%22%20stroke%3D%22%232563eb%22%2F%3E%3Ctext%20x%3D%22277%22%20y%3D%22352%22%20text-anchor%3D%22middle%22%20font-size%3D%2212%22%20fill%3D%22currentColor%22%20font-weight%3D%22600%22%3EBottom%20Half%3C%2Ftext%3E%3Ctext%20x%3D%22277%22%20y%3D%22369%22%20text-anchor%3D%22middle%22%20font-size%3D%2210%22%20fill%3D%22currentColor%22%3ESoftirq%20%EB%93%B1%3C%2Ftext%3E%3C%2Fsvg%3E)
일견 threaded IRQ와 softirq는 상당히 유사하게 느껴진다. 하지만 앞서 언급했듯, softirq는 커널에 이미 정해진 구현이고, 커널을 개조하지 않는 이상 추가할 수는 없다. 따라서 그 맥락도 atomic한 탓에 `schedule`, `sleep` 등은 호출할 수 없다.

threaded IRQ는 그 핸들러 자체를 구현할 수 있기에 관련된 제약이 없다(스핀락을 잡았거나 하는 경우는 별개). 역으로 말하면, 스케줄러의 개입이 필요한 구현의 경우, threaded IRQ를 선택할 수 있다.
# Reference