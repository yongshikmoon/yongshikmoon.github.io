---
layout: post
title: "task_struct - 2"
tags: [linux, kernel, task]
---
# task_struct

***

## exit_state
`exit_state` 필드는 태스크가 종료 상태(`TASK_DEAD`)일 때 가질 수 있는 하위 상태를 나타낸다. 하위 상태는 총 2가지가 있다.

### exit_state의 종료 하위 상태
종료 하위 상태는 다음과 같이 두 가지가 있다. 
1. `EXIT_ZOMBIE`
    * 태스크가 좀비 상태로 최소한의 리소스(`task_struct` 리소스 중 일부)을 들고 있는 상태이다.
    * 최소한의 리소스를 남기는 이유는 부모에게 종료 사실을 알리고 부모가 부모 자신과 관련되어 있는 리소스를 정리할 수 있게 하기 위함이다.

2. `EXIT_DEAD`
    * 실제로 task 관련 리소스가 부모에 의해 수거된 상태이다.
    * 부모와 관련되어 있는 task 리소스는 cgroup, /proc 하위 정보, 그리고 pending signal이다.

### 태스크의 종료 흐름
어떤 태스크가 kill signal(9)에 의해 종료되었다고 가정하자.
1. `do_signal() -> do_exit()` 형태로 수행된다.
2. `do_exit()` 중, `exit_notify()`가 현재 태스크의 상태를 `EXIT_ZOMBIE` 상태로 변경한다.
    * 이 때, 부모가 `SIGCHLD` 시그널을 받을 수 없는 상태(ZOMBIE를 정리할 수 없는 상태)라면 `EXIT_DEAD` 상태로 변경하고, 부모가 아닌 자체적으로 정리하도록 한다. [^1]
3. 이후 `do_exit()`은 마지막 `__schedule(false)`를 부르며 자신의 업무를 종료한다.
4. 부모 프로세스에서는 `sys_wait4()` 시스템콜에 의하여 자식 프로세스의 리소스 정리를 시작한다.
    * `sys_wait4() -> wait_consider_task() -> wait_task_zombie()`로 들어간다.
5. 부모 프로세스는 `wait_task_zombie()`에서 자식 태스크의 상태를 `EXIT_DEAD` 상태로 변경한다.
    * 이후, `release_task()`를 콜하여 부모와 관련되어 있는 task 리소스들을 정리한다.
6. `task_struct` 메모리 자체는 SoftIRQ 핸들러에 의해 정리된다.
    * `task_struct`는 부모가 task 관련 정보를 정리한 뒤, `SoftIRQ` 핸들러로써 `delayed_put_task_struct()`가 실행되어 reference count가 0인 경우 `task_struct`를 해제한다. 
    * 3번 흐름 이후, `context_switch() -> finish_task_switch()`에서도 `task_struct` 해제가 가능하나, 일반적으로 레퍼런스 카운트만 줄이는것으로 보인다.

### 결론
즉, `EXIT_ZOMBIE`는 종료는 되었지만 부모가 리소스를 정리하기 전, 태스크 기능을 할 수 없는 시체(?)같은 태스크 종료 상태라고 보면 될 것 같다.
`EXIT_DEAD`는 태스크가 완전히 정리되어 `__put_task_struct()`에 의해 정리되기 바로 직전 상태라고 보면 될 것 같다.

***

[^1]: 커널 내부에서는 autoreap 이라고 부르는 과정인 것 같다. 하지만 이 부분은 예외 케이스이기 때문에 넘어가도록 한다.