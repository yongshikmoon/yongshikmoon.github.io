---
layout: post
title: "About preempt_count and TIF_NEED_RESCHED, PREEMPT_NEED_RESCHED"
tags: [linux, kernel, task]
---

# preempt_count
`preempt_count`는 `preempt_disable`에 의해 증가되며 `preempt_enable`에 의해 감소되는 `thread_info`내의 `integer`값이다.
즉, `preempt_count`가 0보다 크면 그 태스크는 선점 당하지 않으며, `preempt_count`가 0이면 그 태스크는 선점 될 수 있다.

## TIF_NEED_RESCHED
예측컨데, `thread info need reschedule`의 약자인 것 같다.
현재 이 플래그가 세팅 된 태스크(`thread_info->flags`)는 선점 가능하며, 스케쥴링이 필요하다는 뜻이다.

## PREEMPT_NEED_RESCHED
플래그 이름에서 알 수 있듯이, 이 플래그가 셋팅된 태스크는 선점 가능하며 스케쥴링이 필요하다는 뜻이다.
이 플래그는 `thread_info->preempt_count`에 세팅이 된다.
`PREEMPT_NEED_RESCHED`는 `0x8000000(32bit MSB)`이며, 스케쥴링이 필요한 경우 관련 비트를 클리어 한다.  (**inverted flag bit**)

## TIF_NEED_RESCHED vs PREEMPT_NEED_RESCHED
`thread_info`에 동일한 의미의 플래그가 두 개 있다. 헷갈린다.
관련 내용은 [2013년 패치노트, sched: Add NEED_RESCHED to the preempt_count](https://linux-kernel.vger.kernel.narkive.com/rXhJNpXv/tip-sched-core-sched-add-need-resched-to-the-preempt-count)에 존재한다.

동일한 의미의 플래그가 두 개 있는 이유를 요약하자면 다음과 같다. 
(*이해를 위해 자체 추측 내용 또한 포함되어 있다.*)
1. `TIF_NEED_RESCHED`가 있고, 이 플래그는 리스케쥴링이 필요하다는 플래그였다.
2. 다만, 리스케쥴링을 위해서는 `preempt_count == 0`이여야 했다. 즉, 현 태스크가 선점 가능해야 했다.
    * 주의) 물론 `TIF_NEED_RESCHED`가 셋팅 되어 있다고 해서, `preempt_count == 0`일 필요는 없다.
3. 즉, 스케쥴링을 위해서는 `TIF_NEED_RESCHED`와 `preempt_count`를 **동시에** 체크해야 한다.
    * 이는 critical section이 필요한 작업으로 성능상 불이익을 초래한다.
4. `preempt_count`만을 읽어서 리스케쥴링이 필요한지 체크할 수 있을까?

해결법은 간단하다.
`TIF_NEED_RESCHED` 플래그를 `preempt_count`로 미리 반영하는 것이다.
다만, `TIF_NEED_RESCHED`를 `preempt_count`로 수시로 반영하면 atomic operation이 계속 발생하여 성능상 불이익이 발생한다.
(`preempt_count`를 셋팅할 때, atomic operation으로 모든 코어가 동일한 값을 보도록 해야 한다.)
필요할 때만 반영하도록 리스케쥴링 여부를 체크할 때, 바로 전에 `TIF_NEED_RESCHED`를 `preempt_count`에 반영한다.
(플래그 반영 위치는 위 패치노트에 적혀 있다.)

`TIF_NEED_RESCHED`를 반영함으로써, 스케쥴링이 필요하면 `preempt_count == 0`으로 될 수 있는 가능성[^1]을 만들어 주고, 스케쥴링이 불필요하면 무조건 `preempt_count != 0`으로 만들면 된다.

`PREEMPT_NEED_RESCHED := 0x80000000` 일 때,
스케쥴링이 필요한 경우,
`preempt_count := preempt_count & (~PREEMPT_NEED_RESCHED)`
스케쥴링이 불필요한 경우,
`preempt_count := preempt_count | PREEMPT_NEED_RESCHED`
로 만들어버린다.

이를 통해 `preempt_count`만 체크하여 `preempt_count == 0`이면 리스케쥴링을 수행한다.

*즉, TIF_NEED_RESCHED는 리스케쥴링이 필요할 때 세팅되며, 바로 직전에는 PREEMPT_NEED_RESCHED(inverted)가 세팅된다.*

[^1]: 가능성이라 한 이유는 preempt_count가 0이 아니면 TIF_NEED_RESCHED가 있더라도 리스케쥴 될 수 없기 때문이다.