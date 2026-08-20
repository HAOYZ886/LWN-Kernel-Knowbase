---
title: "Per-vector software-interrupt masking"
url: https://lwn.net/Articles/779738/
date: "February 15, 2019"
category: "Interrupts-Software"
author: "By Jonathan Corbet February 15, 2019"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 15, 2019

Software interrupts (or "softirqs") are one of the oldest deferred-execution mechanisms in the kernel, and that age shows at times. Some developers have been occasionally heard to mutter about removing them, but softirqs are too deeply embedded into how the kernel works to be easily ripped out; most developers just leave them alone. So the recent [per-vector softirq masking patch set](<https://lwn.net/ml/linux-kernel/20190212171423.8308-1-frederic@kernel.org/>) from Frederic Weisbecker is noteworthy as an exception to that rule. Weisbecker is not getting rid of softirqs, but he is trying to reduce their impact and improve their latency. 

Hardware interrupts are the means by which the hardware can gain a CPU's attention to signal the completion of an I/O operation or some other situation of interest. When an interrupt is raised, the currently running code is (usually) preempted and an interrupt handler within the kernel is executed. A cardinal rule for interrupt handlers is that they must execute quickly, since they interfere with the other work the CPU is meant to be doing. That usually implies that an interrupt handler will do little more than acknowledge the interrupt to the hardware and set aside enough information to allow the real processing work to be done in a lower-priority mode. 

The kernel offers a number of deferred-execution mechanisms through which that work can eventually be done. In current kernels, the most commonly used of those is workqueues, which can be used to queue a function call to be run in kernel-thread context at some later time. Another is tasklets, which execute at a higher priority than workqueues; adding new tasklet users tends to be mildly discouraged for reasons we'll get to. Other kernel subsystems might use timers or dedicated kernel threads to get their deferred work done. 

#### Softirqs

Then, there are softirqs which, as their name would suggest, are a software construct; they are patterned after hardware interrupts, but hardware interrupts are enabled while software interrupts execute. Softirqs have assigned numbers ("vectors"); "raising" a particular softirq will cause the handler function for the indicated vector to be called at a convenient time in the near future. That "convenient time" is usually either at the end of hardware-interrupt processing or when a processor that has disabled softirq processing re-enables it. Softirqs thus run outside of the CPU scheduler as a relatively high-priority activity. 

In the 5.0-rc kernel, there are ten softirq vectors defined: 

  * `HI_SOFTIRQ` and `TASKLET_SOFTIRQ` are both for the execution of tasklets; this is part of why tasklets are discouraged. High-priority tasklets are run ahead of any other softirqs, while normal-priority tasklets are run in the middle of the pack. 
  * `TIMER_SOFTIRQ` is for the handling of timer events. `HRTIMER_SOFTIRQ` is also defined; it was once used for high-resolution timers, but that has not been the case since [this change](<https://git.kernel.org/linus/c6eb3f70d448>) was made for the 4.2 release. 
  * `NET_TX_SOFTIRQ` and `NET_RX_SOFTIRQ` are used for network transmit and receive processing, respectively. 
  * `BLOCK_SOFTIRQ` handles block I/O completion events; this functionality was [moved to softirq](<https://git.kernel.org/linus/ff856bad67cb>) mode for the 2.6.16 kernel in 2006. 
  * `IRQ_POLL_SOFTIRQ` is used by the [irq_poll mechanism](<https://elixir.bootlin.com/linux/v5.0-rc6/source/lib/irq_poll.c>), which was [generalized](<https://git.kernel.org/linus/511cbce2ff>) from the block interrupt-polling mechanism for the 4.5 release in 2015. Its predecessor, `BLOCK_IOPOLL_SOFTIRQ` was [added](<https://git.kernel.org/linus/5e605b64a183>) for the 2.6.32 release in 2009; no softirq vectors have been added since then. 
  * `SCHED_SOFTIRQ` is used by the scheduler to perform load-balancing and other scheduling tasks. 
  * `RCU_SOFTIRQ` performs read-copy-update processing. There was [an attempt](<https://git.kernel.org/linus/a26ac2455ffcf3>) made by the late Shaohua Li in 2011 to move this processing to a kernel thread, but performance regressions forced that change to be reverted shortly thereafter. 

Thomas Gleixner once [summarized](<https://lwn.net/Articles/518993/>) the software-interrupt mechanism as ""a conglomerate of mostly unrelated jobs, which run in the context of a randomly chosen victim w/o the ability to put any control on them"". For historical reasons that long predate Linux, software interrupts also sometimes go by the name "bottom halves" — they are the half of interrupt processing that is done outside of hardware interrupt mode. For this reason, one will often see the term "BH" used to refer to software interrupts. 

Since software interrupts execute at a high priority, they can create high levels of latency in the system if they are not carefully managed. As little work as possible is done in softirq mode, but certain kinds of system workloads (high network traffic, for example) can still cause softirq processing to adversely impact the system as a whole. The kernel will actually kick softirq handling out to a set of `ksoftirqd` kernel threads if it starts taking too much time, but there can be performance costs even if the total CPU time used by softirq processing is relatively low. 

#### Softirq concurrency

Part of the problem, especially for latency-sensitive workloads, results from the fact that softirqs are another source of concurrency in the system that must be controlled. Any work that might try to access data concurrently with a softirq handler must use some sort of mutual exclusion mechanism and, since softirqs are essentially interrupts, special care must be taken to avoid deadlocks. If, for example, a kernel function acquires a spinlock, but is then interrupted by a softirq that tries to take the same lock, that softirq handler will wait forever — the sort of situation that latency-sensitive users tend to get especially irritable over. 

To avoid such problems, the kernel provides a number of ways to prevent softirq handlers from running for a period of time. For example, a call to `spin_lock_bh()` will acquire the indicated spinlock and also disable softirq processing for as long as the lock is held, preventing the deadlock scenario described above. Any subsystem that uses software interrupts must take care to ensure that they are disabled in places where unwanted concurrency could occur. 

Linux software interrupts have an interesting problem — interesting because it is seemingly obvious but has been there since the beginning. The softirq vectors described above are all independent of each other, and their handlers are unlikely to interfere with each other. Network transmit processing should not be bothered if the block softirq handler runs concurrently, for example. So code that must protect against concurrent access from a softirq handler need only disable the one handler that it might race with, but functions like `spin_lock_bh()` disable _all_ softirq handling. That can cause unrelated handlers to be delayed needlessly, once again leading to bad temper in the low-latency camp. 

#### Per-vector masking

Weisbecker's answer to this is to allow individual softirq vectors to be disabled while the others remain enabled. The [first attempt](<https://lwn.net/ml/linux-kernel/1539213137-13953-1-git-send-email-frederic@kernel.org/>), posted in October 2018, changed the prototypes of functions like `spin_lock_bh()`, `local_bh_disable()`, and `rcu_read_lock_bh()` to contain a mask of the vectors to disable. There was just one little problem: there are a lot of callers to those functions in the kernel. So the bottom line for that patch set was: 

```
945 files changed, 13857 insertions(+), 9767 deletions(-)
```

The kernel community has gotten good at merging large, invasive patch sets, but that one still pushed the limits a bit. That is especially true given that almost all call sites still disabled all vectors; doing anything else requires careful auditing of every change. The second time around, Weisbecker decided to take an easier approach and define new functions, leaving the old ones unchanged. So this patch set introduces functions like: 

```
unsigned int spin_lock_bh_mask(spinlock_t *lock, unsigned int mask);
        unsigned int local_bh_disable_mask(unsigned int mask);
        /* ... */
```

After the call, only the softirq vectors indicated by the given `mask` will have been disabled; the rest can still be run if they were enabled before the call. The return value of these functions is the previous set of masked softirqs; it is needed when renabling softirqs to their previous state. 

This patch set is rather less intrusive: 

```
36 files changed, 690 insertions(+), 281 deletions(-)
```

That is true even though it goes beyond the core changes to, for example, add support to the lockdep locking checker to ensure that the use of the vector masks is consistent. One thing that has not yet been done is to allow one softirq handler to preempt another; that's on the list for future work. 

No performance numbers have been provided, so it is not possible to know for sure that this work has achieved its goal of providing better latencies for specific softirq handlers. Still, networking maintainer David Miller [indicated his approval](<https://lwn.net/ml/linux-kernel/20190212.102912.1498886718723769885.davem@davemloft.net/>), saying: ""I really like this stuff, nice work"". Linus Torvalds had some low-level comments that will need to be addressed in the next iteration of the patch set. Some other important reviewers have yet to weigh in, so it would be too soon to say that this work is nearly ready. But, in the absence of a complete removal of softirqs, there is clear value in not disabling them needlessly, so this change seems likely to vector itself into the mainline sooner or later.
