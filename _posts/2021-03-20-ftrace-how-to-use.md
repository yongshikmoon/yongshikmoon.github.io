---
layout: post
title: "About ftrace"
---

#About ftrace

## ftrace란?
리눅스 커널 개발자에게 커널 내부에서 어떤 함수가 불렸는지, 어떤 이벤트가 발생했는지 알려주는 트레이스 기능이다.
예를 들어, 목표 커널 함수[^1]가 불릴 때 콜스택을 확인할 수 있다.
콜스택을 확인함으로써 목표 커널 함수가 어떤 함수에 의해 불리는지 확인할 수 있다. (커널 공부에 좋을듯)

***

## ftrace 사용법
### 1. 트레이스 포인트 지정하기
임의의 커널 함수 foo()에 진입 할 때의, 콜스택을을 확인하고 싶은 경우 아래와 같은 명령어를 입력한다.
```bash
$ echo 0 > /sys/kernel/debug/tracing/tracing_on  # trace off
$ echo secondary_start_kernel > /sys/kernel/debug/tracing/set_ftrace_filter # secondary_start_kernel에 트레이스 포인트 설정
$ echo function > /sys/kernel/debug/tracing/current_tracer # tracer로 function 지정
$ echo foo > /sys/kerenl/debug/tracing/set_ftrace_filter # foo에 트레이스 포인트 설정
$ echo 1 > /sys/kernel/debug/tracing/options/func_stack_trace # 함수 entry에서 콜스택을 출력하도록 설정 <stack trace>
$ echo 1 > /sys/kernel/debug/tracing/options/sym-offset # 콜스택에 호출 오프셋 출력하도록 설정 예) vfs_read+0x9c
$ echo 1 > /sys/kernel/debug/tracing/tracing_on # trace on
```

특이한 부분에 대한 설명을 간략히 하고 넘어가면,
```bash
$ echo secondary_start_kernel > /sys/kernel/debug/tracing/set_ftrace_filter # secondary_start_kernel에 트레이스 포인트 설정
```
`set_ftrace_filter`에 아무것도 설정하지 않고 ftrace를 키면, ftrace는 모든 커널 함수에 대하여 트레이싱을 한다.
모든 커널 함수에 의해 트레이스가 발생되면, 그 오버헤드가 엄청나 시스템은 락업 상태에 빠진다.
그러므로 부팅 이후 절대 불리지 않을 함수`secondary_start_kernel`[^2]를 트레이스 포인트로 찍어준다.
```bash
$ echo function > /sys/kernel/debug/tracing/current_tracer # tracer로 function 지정
```
`function`은 함수 호출(function entry)시, 원하는 정보를 찍는 트레이서이다.
그 외로는 `function_tracer`가 있으며 `function`과 비슷하나 function exit을 표현한다는 점이 다르다.
어떠한 트레이서도 설정하지 않으려면, `nop`을 지정한다.

### 2. 트레이스 뽑기
트레이스를 작동 시킨 후, 커널 함수 `foo`를 목표 커널함수로 설정 하고나서, 트레이스를 뽑아본다. 
뽑힌 트레이스는 메모리 링버퍼에 저장되어 있으며, 링버퍼의 내용을 읽어 파일로 저장한다.
```bash
$ echo 0 > /sys/kernel/debug/tracing/tracing_on # trace off
$ cp /sys/kernel/debug/tracing/trace trace_log # 링버퍼의 내용을 trace_log로 저장
```

### 3. function 트레이스 내용 분석
```bash
cat-2834  [003] d...  7890.883529: rpi_get_interrupt_info+0x14/0x6c <-show_interrupts+0x2e0/0x3e4
cat-2834  [003] d...  7890.883574: <stack trace>
 => rpi_get_interrupt_info+0x18/0x6c
 => show_interrupts+0x2e0/0x3e4
 => seq_read+0x388/0x4d4
 => proc_reg_read+0x70/0x98
 => __vfs_read+0x48/0x16c
 => vfs_read+0x9c/0x164
 => ksys_read+0x74/0xe8
 => sys_read+0x18/0x1c
 => ret_fast_syscall+0x0/0x28
 => 0x7e990520
```
먼저 `d...`의 의미를 살펴보면 다음과 같다.

`d`: interrupt disabled (현재 컨텍스트에서의 인터럽트 허용 플래그와 같다.)

`n`: 'N' both TIF_NEED_RESCHED & PREEMPT_NEED_RESCHED 'n' for TIF_NEED_RESCHED p for 'PREEMPT_NEED_RESCHED'[^3]

`h/s`: interrupt context or softIRQ context

`0~5`: preempt_count

`thread_info`의 `preempt_count`의 각 필드를 참조하여 출력하는 것으로 추정된다.

![lwn.net](https://static.lwn.net/images/2020/preempt-count.svg)[^4]

그 외적으로 특이한 부분은 없는 것 같다.

### 4. Events 트레이스 내용 분석
Events는 커널에서 미리 정의되어있는 이벤트에 대하여 트레이스를 뽑게 도와준다.
각 이벤트는 미리 정의된 포맷으로 출력되며, 각 함수 기능에 따라 충분히 도움이 되는 정보를 제공한다.
예를 들어, IRQ의 경우 IRQ 번호, IRQ 핸들링 성공 여부 등을 출력한다. 그리고 sched_switch의 경우 이전 태스크와 다음 태스크의 정보를 출력해 준다.
아래는 events 트레이스를 ON 시키기 위한 명령어 들이다.
```bash
$ echo 0 > /sys/kernel/debug/tracing/tracing_on  # trace off
$ echo 0 > /sys/kernel/debug/tracing/events/enable # ALL event traces off
$ echo 1 > /sys/kernel/debug/tracing/events/sched/sched_switch/enable
$ echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_entry/enable
$ echo 1 > /sys/kernel/debug/tracing/events/irq/irq_handler_exit/enable
$ echo 1 > /sys/kernel/debug/tracing/tracing_on # trace on
```
이후, **트레이스 뽑기**에서 나와있는 방법으로 트레이스를 뽑는다.
결과는 다음과 같다.
```
    kworker/u8:1-3871  [003] d... 33794.390123: sched_switch: prev_comm=kworker/u8:1 prev_pid=3871 prev_prio=120 prev_state=S ==> next_comm=kworker/u8:1 next_pid=4010 next_prio=120
          <idle>-0     [000] d.h. 33794.391185: irq_handler_entry: irq=162 name=arch_timer
          <idle>-0     [000] dnh. 33794.391219: irq_handler_exit: irq=162 ret=handled
```
위에서 `irq_handler_entry`와 `irq_handler_exit`에 의해 발생한 트레이스부터 보면 `irq 162`이 발생하여 `arch_timer`라는 irq handler를 실행하였고, 결과는 제대로 핸들링(`handled`)되었다는것을 알 수 있다.
또한 `sched_switch`에 의해 발생한 트레이스는 `kworker(pid: 3871)`로부터 다른 `kworker(pid: 4010)`으로 컨텍스트 스위칭이 일어났다는것을 알 수 있다.
그 외에도 정말 많은 커널 이벤트들을 트레이스로 뽑을 수 있다.
뽑을 수 있는 기 정의된 이벤트들은 `/sys/kernel/debug/tracing/events/`를 보면 알 수 있다.

***

## ftrace의 event는 어떻게 발생되어 기록되는가?
모든 이벤트 트레이스는 `trace_ + event_name` 함수를 호출하여 기록된다.
예를 들어, `irq_handler_entry`는 `trace_irq_handler_entry` 함수를 이용하여 트레이스가 기록된다.
### 1. irq_handler_entry & irq_handler_exit
```cpp
/* code at kernel/irq/handle.c */
trace_irq_handler_entry(irq, action);
res = action->handler(irq, action->dev_id);
trace_irq_handler_exit(irq, action, res);
```
위 코드에서 보듯이, IRQ 핸들러 실행 전후에 트레이스 기록을 수행한다.
위 함수들은 아래에 다음과 같이 매크로 형태로 정의되어 있다.
```cpp
TRACE_EVENT(irq_handler_entry,
    TP_PROTO(int irq, struct irqaction *action),
    TP_ARGS(irq, action),
    TP_STRUCT__entry(
        __field(    int,    irq     )
        __string(   name,   action->name    )
    ),
    TP_fast_assign(
        __entry->irq = irq;
        __assign_str(name, action->name);
    ),
    TP_printk("irq=%d name=%s", __entry->irq, __get_str(name))
);
```
### 2. sched_switch
```cpp
static void __sched notrace __schedule(bool preempt)
{
    ...
    if (likely(prev != next)) {
        rq->nr_switches++;
        rq->curr = next;
        ++*switch_count;

        trace_sched_switch(preempt, prev, next);

        /* Also unlocks the rq: */
        rq = context_switch(rq, prev, next, &rf);
    } else {
    ...
}
```
위에서 `task_struct *prev`에서 `task_struct* next`로 `context_switch()`가 발생할 때, `trace_sched_switch()`가 호출됨을 알 수 있다.
`trace_schecd_switch()` 내부를 보면 `task state`가 어떻게 정의되어 있는지 유추가 가능하다.
```cpp
    TP_printk("prev_comm=%s prev_pid=%d prev_prio=%d prev_state=%s%s ==> next_comm=%s next_pid=%d next_prio=%d",
        __entry->prev_comm, __entry->prev_pid, __entry->prev_prio,

        (__entry->prev_state & (TASK_REPORT_MAX - 1)) ?
          __print_flags(__entry->prev_state & (TASK_REPORT_MAX - 1), "|",
                { TASK_INTERRUPTIBLE, "S" },
                { TASK_UNINTERRUPTIBLE, "D" },
                { __TASK_STOPPED, "T" },
                { __TASK_TRACED, "t" },
                { EXIT_DEAD, "X" },
                { EXIT_ZOMBIE, "Z" },
                { TASK_PARKED, "P" },
                { TASK_DEAD, "I" }) :
          "R",

        __entry->prev_state & TASK_REPORT_MAX ? "+" : "",
        __entry->next_comm, __entry->next_pid, __entry->next_prio)
```
```    
kworker/u8:1-3871  [003] d... 33794.390123: sched_switch: prev_comm=kworker/u8:1 prev_pid=3871 prev_prio=120 prev_state=S ==> next_comm=kworker/u8:1 next_pid=4010 next_prio=120
```
위에서는 `prev_state=S`이므로, `TASK_INTERRUPTIBLE` 상태임을 알 수 있다.

***

>**본 글은 "디버깅을 통해 배우는 리눅스 커널의 구조와 원리"를 참조하였습니다.**

***

[^1]: trace가 가능한 커널 함수는 /sys/kernel/debug/tracing/available_tracers로 확인할 수 있다. 그 외의 함수를 트레이싱 하려고 하면 시스템이 다운될 수 있다.
[^2]: `secondary_start_kernel` 함수는 부트 시퀀스에서 CPU 0가 아닌 다른 CPU를 부팅할 때 쓰는 함수이며, 부팅 이후는 절대 부를 일이 없는 함수이다.
[^3]: 정확하지는 않으나, TIF_NEED_RESCHED가 세팅된 후, PREEMPT_NEED_RESHCED로 전파되어 실제 reschedule 발생시에는 PREEMPT_NEED_RESCHED를 참조한다.. 라고 쓰여있으나, 코드내에서는 직접적으로 참조하는 부분은 없는것 같다.. 발견하면 수정 예정
[^4]: [Four short stories about preempt_count()](https://lwn.net/Articles/831678/)