---
title: The long road to lazy preemption
url: https://lwn.net/Articles/994322/
date: "October 18, 2024"
category: "Preemption; Scheduler-Preemption models"
author: "By Jonathan Corbet October 18, 2024"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 18, 2024

The kernel's CPU scheduler currently offers several preemption modes that implement a range of tradeoffs between system throughput and response time. Back in September 2023, a [discussion on scheduling](<https://lwn.net/Articles/944686/>) led to the concept of "lazy preemption", which could simplify scheduling in the kernel while providing better results. Things went quiet for a while, but lazy preemption has returned in the form of [this patch series](<https://lwn.net/ml/all/20241007074609.447006177@infradead.org>) from Peter Zijlstra. While the concept appears to work well, there is still a fair amount of work to be done. 

#### Some review

Current kernels have four different modes that regulate when one task can be preempted in favor of another. `PREEMPT_NONE`, the simplest mode, only allows preemption to happen when the running task has exhausted its time slice. `PREEMPT_VOLUNTARY` adds a large number of points within the kernel where preemption can happen if needed. `PREEMPT_FULL` allows preemption at almost any point except places in the kernel that prevent it, such as when a spinlock is held. Finally, `PREEMPT_RT` prioritizes preemption over most other things, even making most spinlock-holding code preemptible. 

A higher level of preemption enables the system to respond more quickly to events; whether an event is the movement of a mouse or an "imminent meltdown" signal from a nuclear reactor, faster response tends to be more gratifying. But a higher level of preemption can hurt the overall throughput of the system; workloads with a lot of long-running, CPU-intensive tasks tend to benefit from being disturbed as little as possible. More frequent preemption can also lead to higher lock contention. That is why the different modes exist; the optimal preemption mode will vary for different workloads. 

Most distributions ship kernels built with the `PREEMPT_DYNAMIC` pseudo-mode, which allows any of the first three modes to be selected at boot time, with `PREEMPT_VOLUNTARY` being the default. On systems with debugfs mounted, the current mode can be read from `/sys/kernel/debug/sched/preempt`. 

`PREEMPT_NONE` and `PREEMPT_VOLUNTARY` do not allow the arbitrary preemption of code running in the kernel; there are times when that can lead to excessive latency even in systems where minimal latency is not prioritized. This problem is the result of places in the kernel where a large amount of work can be done; if that work is allowed to run unchecked, it can disrupt the scheduling of the system as a whole. To get around this problem, long-running loops have been sprinkled with calls to `cond_resched()`, each of which is an additional voluntary preemption point that is active even in the `PREEMPT_NONE` mode. There are hundreds of these calls in the kernel. 

There are some problems with this approach. `cond_resched()` is a form of heuristic that only works in the places where a developer has thought to put it. Some calls are surely unnecessary, while there will be other places in the kernel that could benefit from `cond_resched()` calls, but do not have them. The use of `cond_resched()`, at its core, takes a decision that should be confined to the scheduling code and spreads it throughout the kernel. It is, in short, a bit of a hack that mostly works, but which could be done better. 

#### Doing better

The tracking of whether a given task can be preempted at any moment is a complicated affair that must take into account several variables; see [this article](<https://lwn.net/Articles/945422/>) and [this article](<https://lwn.net/Articles/831678/>) for details. One of those variables is a simple flag, `TIF_NEED_RESCHED`, that indicates the presence of a higher-priority task that is waiting for access to the CPU. Events such as waking a high-priority task can cause that flag to be set in whatever task is currently running. In the absence of this flag, there is no need for the kernel to consider preempting the current task. 

There are various points where the kernel can notice that flag and cause the currently running task to be preempted. The scheduler's timer tick is one example; any time a task returns to user space from a system call is another. The completion of an interrupt handler is yet another, but that check, which can cause preemption to happen any time that interrupts are enabled, is only enabled in `PREEMPT_FULL` kernels. A call to `cond_resched()` will also check that flag and, if it is set, call into the scheduler to yield the CPU to the other task. 

The lazy-preemption patches are simple at their core; they add another flag, `TIF_NEED_RESCHED_LAZY`, that indicates a need for rescheduling at some point, but not necessarily right away. In the lazy preemption mode (`PREEMPT_LAZY`), most events will set the new flag rather than `TIF_NEED_RESCHED`. At points like the return to user space from the kernel, either flag will lead to a call into the scheduler. At the voluntary preemption points and in the return-from interrupt path, though, only `TIF_NEED_RESCHED` is checked. 

The result of this change is that, in lazy-preemption mode, most events in the kernel will not cause the current task to be preempted. That task _should_ be preempted eventually, though. To make that happen, the kernel's timer-tick handler will check whether `TIF_NEED_RESCHED_LAZY` is set; if so, `TIF_NEED_RESCHED` will also be set, possibly causing the running task to be preempted. Tasks will generally end up running for something close to their full time slice unless they give up the CPU voluntarily, which should lead to good throughput. 

With these changes, the lazy-preemption mode can, like `PREEMPT_FULL`, run with kernel preemption enabled at (almost) all times. Preemption _can_ happen any time that the preemption counter says that it should. That allows long-running kernel code to be preempted whenever other conditions do not prevent it. It also allows preemption to happen quickly in those cases where it is truly needed. For example, should a realtime task become runnable, as the result of handling an interrupt, for example, the `TIF_NEED_RESCHED` flag will be set, leading to an almost immediate preemption. There will be no need to wait for the timer tick in such cases. 

Preemption will _not_ happen, though, if only `TIF_NEED_RESCHED_LAZY` is set, which will be the case much of the time. So a `PREEMPT_LAZY` kernel will be far less likely to preempt a running task than a `PREEMPT_FULL` kernel. 

#### Removing `cond_resched()` — eventually

The end goal of this work is to have a scheduler with only two non-realtime modes: `PREEMPT_LAZY` and `PREEMPT_FULL`. The lazy mode will occupy a place between `PREEMPT_NONE` and `PREEMPT_VOLUNTARY`, replacing both of them. It will, however, not need the voluntary preemption points that were added for the two modes it replaces. Since preemption can now happen almost anywhere, there is no longer a need to enable it in specific spots. 

For now, though, the `cond_resched()` calls remain; if nothing else, they are required for as long as the `PREEMPT_NONE` and `PREEMPT_VOLUNTARY` modes exist. Those calls also help to ensure that problems are not introduced while lazy preemption is being stabilized. 

In the current patch set, `cond_resched()` only checks `TIF_NEED_RESCHED`, meaning that preemption will be deferred in many situations where it will happen immediately from `cond_resched()` in `PREEMPT_VOLUNTARY` or `PREEMPT_NONE` mode. Steve Rostedt [questioned](<https://lwn.net/ml/all/20241009100133.2569e2a7@gandalf.local.home>) this change, asking whether `cond_resched()` should retain its older meaning, at least for the `PREEMPT_VOLUNTARY` case. Even though `PREEMPT_VOLUNTARY` is slated for eventual removal, he thought, keeping the older behavior could help to ease the transition. 

Thomas Gleixner [answered](<https://lwn.net/ml/all/87h69lqbk0.ffs@tglx>) that only checking `TIF_NEED_RESCHED` is the correct choice, since it will help in the process of removing the `cond_resched()` calls entirely: 

> That forces us to look at all of them and figure out whether they need to be extended to include the lazy bit or not. Those which do not need it can be eliminated when LAZY is in effect because that will preempt on the next possible preemption point once the non-lazy bit is set in the tick. 

He added that he expects ""less than 5%"" of the `cond_resched()` calls need to check `TIF_NEED_RESCHED_LAZY` and, thus, will need to remain even after the transition to `PREEMPT_LAZY` is complete. 

Before then, though, there are hundreds of `cond_resched()` calls that need to be checked and, for most of them at least, removed. Many other details have to be dealt with as well; [this patch set](<https://lwn.net/ml/all/20241009165411.3426937-1-ankur.a.arora@oracle.com>) from Ankur Arora addresses a few of them. There is also, of course, the need for extensive performance testing; Mike Galbraith has made [an early start](<https://lwn.net/ml/all/579b7ea34ef6e2f7c955abdfc0929fe1af36faef.camel@gmx.de>) on that work, showing that throughput with lazy preemption falls just short of that with `PREEMPT_VOLUNTARY`. 

It all adds up to a lot to be done still, but the end result of the lazy-preemption work should be a kernel that is a bit smaller and simpler while delivering predictable latencies without the need to sprinkle scheduler-related calls throughout the code. That seems like a better solution, but getting there is going to take some time.
