---
layout: post
title: "task_struct - 1"
tags: [linux, kernel, task]
---
# task_struct

***
## state
`state` 필드는 task의 실행 상태를 나타내며, `exit_state`[^0]는 task의 종료 상태를 나타낸다.  
사실 `state`와 `exit_state`는 하나의 필드로 합쳐져있는것으로 보이나[^2], 실수를 줄이기 위해 실행 상태와 종료 상태를 분리한것으로 보인다.  


### task의 실행 상태 (state)
태스크의 실행 상태는 다음의 값을 가질 수 있다. (kernel 4.19 기준)   
이 때, 4, 5, 6, 7은 speical task state로 불린다.  
1. `TASK_RUNNING`
    * task가 정상적으로 실행되고 있을 때를 뜻한다. 실행이라 함은 CPU에서 직접 실행하고 있는 중, 그리고 런큐에서 CPU를 받기 위해 대기하는 중을 포함한다.
2. `TASK_INTERRUPTIBLE`
    * task가 어떠한 이유(예, 사용자 입력 대기)로 blocking되었음을 뜻한다. 원인이 되는 이유가 해결되면 그 task는 깨어난다. (예, 인풋을 기다리는 SSH) 또는 이 task로 signal이 새로 들어오면, 그 signal을 처리해야 한다.
    * Signal 처리 가능 여부가 상당히 중요한 요소인데, 다음과 같은 예를 보자.
    * 커널 레벨에서 task는 blocking 되어야 할 이유가 있으면 `wait_event_interruptible()`을 호출하고, task를 `TASK_INTERRUPTIBLE` 상태로 만든다. 이 상태에서 task에게 시그널(예, SIGKILL)이 들어오면 `wait_event_interruptible()`은 `-ERESTARTSYS`을 반환한다.[^3] 즉, blocking 이유가 해결되지 않은 상태에서 task가 깨어나고, 그 task는 signal handling을 마저 수행해야 한다. 그리고 다시 task는 `wait_event_interruptible()`를 호출하여 blocking 이유가 없어질때까지 다시 blocking 되어야 한다.
3. `TASK_UNINTERRUPTIBLE`
    * 보통 I/O operation과 같이 `TASK_RUNNING`에 있어 중요한 작업을 처리하기 위한 상태이다.
    * 위와 다르게 task가 signal에 의해 깨어날 수 없다[^4]. 명시적인 wakeup 계열 함수(`completion_done()` 등) 외에는 절대 깨어나지 않는다. 만약 task가 강제 종료해야 한다고 하더라도 SIGKILL을 받을 수 없기 때문에 종료할 수 없다.
4. `__TASK_STOPPED`
    * `SIGSTOP` 등의 의해 태스크 실행 중단이 되었음을 뜻한다. 주로 `TASK_WAKEKILL`과 조합하여 많이 쓰인다. (`TASK_STOPPED`)
5. `__TASK_TRACED`
    * `ptrace`에 의해 task가 trace되고 있음을 뜻한다. sys_ptrace로 target task에 attach하면 task state에 `__TASK_TRACED`를 적용한다. 주로 `TASK_WAKEKILL`과 조합하여 많이 쓰인다. (`TASK_TRACED`)
6. `TASK_PARKED`
    * 커널 쓰레드의 parking/unparking과 관련된 상태이다.[^5] 
    * CPU 핫플러그 기능에서 per-cpu kthread(예, ksoftirqd)는 해당 CPU 오프라인 시, `TASK_UNINTERRUPTIBLE`상태로 들어간다. 이 때, `TASK_PARKED`도 같이 표기해 준다. 추후 CPU가 다시 온라인이 되면, per-cpu kthread 태스크는 `TASK_RUNNING`으로 복구 시킬 때, 상태가 `TASK_PARKED`임을 확인하며, 그 TASK를 해당하는 CPU로 재-바인딩 시켜준다.
7. `TASK_DEAD`
    * TASK가 종료할 때, 종료했음을 의미하는 상태이다. 이 상태는 태스크 생존에 필수적이지 않은 모든 리소스(file descriptor 등)를 반환한 상태이며, 다른 태스크로 컨텍스트 스위칭 후, 이 태스크의 스택과 `task_struct` 또한 정리될 예정임을 뜻한다.
8. `TASK_WAKEKILL`
    * `SIGKILL`을 받을 수 있는 상태임을 뜻한다. 이 상태는 다른 상태와 조합하여 쓰인다. 예를 들어, `TASK_UNINTERRUPTIBLE | TASK_WAKEKILL = TASK_KILLABLE` 임을 뜻하며, 다른 signal은 무시하지만 kill 시그널은 무시하지 않는다라는 뜻이다.
9. `TASK_WAKING`
    * Task가 깨어나는 중임을 뜻한다. 
    * 기존의 목적: 두 개 이상의 태스크가 타겟 태스크를 깨우려고 할 때, 한 개 태스크만이 깨울 수 있도록 만든다. [^6]
    * 코드를 읽어보면 딱히 그 목적보다, Task를 깨워서 다른 run queue로 enqueue 하기 전, 원하는 데이터를 재정비 할 수 있도록 상태를 저장해주는 느낌이다. (예, migration이 필요한 경우, task의 virtual runtime을 run queue의 virtual runtime base에 맞게 재조정 해준다.)
10. `TASK_NOLOAD`
    * 이 상태를 할당받은 task는 load를 계산할 때, load 계산에 들어가지 않는다. 일반적으로 `TASK_UNINTERRUPTIBLE | TASK_NOLOAD`로 같이 쓰일 것이라고 설계자는 주장하고 있다. (코드에 잘 보이진 않는다.) [^7]
    * load 계산 시, activate task를 고려하며, activate task는 `TASK_RUNNING`과 `TASK_UNINTERRUPTIBLE`을 포함한다.
    * `TASK_UNINTERRUPTIBLE` 상태는 `TASK_RUNNING` 중, I/O operation과 같이 중요한 일 때문에 불가피하게 blocking 상태로 들어갔음을 의미하며, blocking이 풀리면 바로 `TASK_RUNNING` 상태로 복귀할 것을 기대한다. 이 때문에 load로 계산한다.
    * `TASK_INTERRUPTIBLE`은 사용자 입력 대기 상태 등, 긴 시간동안 휴면 상태로 있어야 하는 태스크가 갖는 상태이며 load로 계산하지 않는다. (일반적으로 긴 시간동안 유휴하기 때문이다.)
11. `TASK_NEW`
    * 태스크가 새롭게 생성되고 있음을 의미한다. `wake_up_new_task()`에서 `TASK_RUNNING`을 할당 받기 전, `copy_process()`의 `sched_fork()` 함수에서 세팅되는 상태이며, **절대 실행되지 않고, 시그널 핸들링도 처리하지 않고, 런큐에 들어갈 수도 없는** 상태를 담보한다.

***

[^0]: https://yongshikmoon.github.io/2021/03/29/task_struct-2.html
[^1]: https://en.wikipedia.org/wiki/Thread_control_block
[^2]: 실제로 state와 exit_state는 비트 플래그가 겹치는 부분이 없다. 즉, 서로 상호배타적인 비트플래그를 갖는다.
[^3]: RESTARTSYS라는 의미를 알 수 있듯이, syscall을 restart하라는 말이다. 다시 말하면 signal handling을 한 후, syscall을 restart하여 `wait_event_interruptible()`을 재실행하라는 의미이며, 이 때문에 syscall은 reenterancy가 있어야 한다라고 말한다.
[^4]: `signal_wake_up()`함수를 보면 TASK_INTERRUPTIBLE state만 signal을 받아 깨어날 수 있음을 알 수 있다.
[^5]: https://lwn.net/Articles/500338/, 구현은 많이 변했지만 주석은 읽어볼만하다.
[^6]: https://lore.kernel.org/patchwork/patch/170913/
[^7]: https://lore.kernel.org/lkml/alpine.LFD.2.11.1505112154420.1749@ja.home.ssi.bg/T/