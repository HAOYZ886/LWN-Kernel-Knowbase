---
title: "Revisiting the kernel's preemption models (part 1)"
url: https://lwn.net/Articles/944686/
date: "September 21, 2023"
category: "Preemption; Scheduler-Preemption models"
author: "By Jonathan Corbet September 21, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 21, 2023

All that Ankur Arora seemingly wanted to do with [this patch set](<https://lwn.net/ml/linux-kernel/20230830184958.2333078-1-ankur.a.arora@oracle.com/>) was to make the process of clearing huge pages on x86 systems go a little faster. What resulted was an extensive discussion on the difficulties of managing preemption correctly in the kernel. It may be that some changes will come to the plethora of preemption models that the kernel currently offers. 

#### Fast memory clearing

The patch set in question adds a function to use x86 string instructions to clear large amounts of memory more quickly; the change produces some nice performance improvements for situations (such as page-fault handling) where ranges of memory must be zeroed out before being given to the faulting process. But there is one little problem with this approach: being able to clear large ranges with a single instruction is nice, but that one instruction can execute for a long time — long enough to create unwanted latencies for other processes running on the same CPU. 

Excess latency caused by long-running operations is not a new problem for the kernel. The usual response is to break those operations up, inserting `cond_resched()` calls to voluntarily give up the CPU for a bit if a higher-priority process needs to run. It is, however, not possible to insert such calls into a single, long-running instruction, so some other mitigation is needed. Arora chose to [add a new task flag](<https://lwn.net/ml/linux-kernel/20230830184958.2333078-8-ankur.a.arora@oracle.com/>) (`TIF_ALLOW_RESCHED`) marking the current task as being preemptible. If the kernel, at the end of handling an interrupt, sees that flag, it knows that it can switch to a higher-priority task if need be. This new flag could be set before clearing the pages, then reset afterward. 

This mechanism turned out to have some problems. The code setting the flag may be preemptible, but other functions it calls may not be. Other events, such as hardware interrupts or CPU traps (page faults, for example) could put the kernel into a non-preemptible situation as well. Having a flag set that marks the current task as being preemptible anyway is just not going to lead to good things. 

It looks like making this idea work with current kernels would require moving away from a task flag, and toward marking specific ranges of code as being preemptible in this way. That could limit how widely this feature could be used, though, since finding out whether the current location is preemptible would require maintaining a data structure of the preemptible ranges and searching it, in the interrupt path, to see if preemption is possible. That made Linus Torvalds [a little unhappy](<https://lwn.net/ml/linux-kernel/CAHk-=whMkp68vNxVn1H3qe_P7n=X2sWPL9kvW22dsvMFH8FcQQ@mail.gmail.com/>): 

> I really hate that, because I was hoping we'd be able to use this to not have so many of those annoying and random "cond_resched()" calls. [...] I was hoping that we'd have some generic way to deal with this where we could just say "this thing is reschedulable", and get rid of - or at least not increasingly add to - the cond_resched() mess. 

Peter Zijlstra [pointed out](<https://lwn.net/ml/linux-kernel/20230911150410.GC9098@noisy.programming.kicks-ass.net/>) that Torvalds was describing full preemption, which the kernel already supports quite well. That led to a bit of a shift in the discussion. 

#### Preemption models

The traditional Unix model does not allow for preemption of the kernel at all; once the kernel gets the CPU, it keeps executing until it voluntarily gives the CPU up. In the beginning, Linux followed this model as well; over the years, though, the kernel has gained a number of preemption modes, selectable with a configuration option: 

  * `PREEMPT_NONE` is the traditional model, with no preemption at all. The kernel must give up the CPU, via a return to user space, a blocking operation, or a `cond_resched()` call, before another task can run. 
  * `PREEMPT_VOLUNTARY` increases (significantly) the number of points where the kernel is said to be voluntarily giving up the CPU. Each call to `might_sleep()`, which is otherwise a debugging function marking functions that could block, becomes a preemption point, for example. 
  * `PREEMPT` is full preemption; the kernel can be preempted at any point where some factor (such as holding a spinlock or explicitly disabling preemption) does not explicitly prevent it. 
  * `PREEMPT_RT` is the realtime preemption mode, where even most spinlocks become preemptible and a number of other changes are made as well. 

These options represent different tradeoffs between throughput and latency. Preemption is not free; it can worsen cache behavior, and tracking the state needed to know whether preemption is safe at any given time has costs of its own. But latency hurts as well, especially for interactive use. At the `PREEMPT_NONE` end of the scale, only throughput matters, and latencies can be long. As the level of preemption increases, latency is reduced, but throughput might suffer as well. 

As an extra complication, another option, `PREEMPT_DYNAMIC`, was [added to the 5.12 kernel](<https://git.kernel.org/linus/6ef869e06474>) by Michal Hocko in 2021. It allows the preemption choice to be deferred until boot time, where any of the modes except `PREEMPT_RT` can be selected by the `preempt=` command-line parameter. `PREEMPT_DYNAMIC` allows distributors to ship a single kernel while letting users pick the preemption mode that works best for their workload. 

Torvalds, seemingly looking closely at `PREEMPT_DYNAMIC` for the first time, [observed](<https://lwn.net/ml/linux-kernel/CAHk-=whc0Jsji_h-vLyDLT6Q-NCjSTMOg9qXACmoPZOdWqMovg@mail.gmail.com/>) that it maintains all of the information about whether the current task is preemptible, even when running in the no-preemption modes. As Zijlstra [responded](<https://lwn.net/ml/linux-kernel/20230912072022.GA35261@noisy.programming.kicks-ass.net/>), that suggests that the overhead of maintaining that information is not seen as being a problem; Ingo Molnar [added](<https://lwn.net/ml/linux-kernel/ZQAVho72j1zG%2FHhK@gmail.com/>) that, while it might be nice to patch that overhead out, ""it's little more than noise on most CPUs, considering the kind of horrible security-workaround overhead we have on almost all x86 CPU types"". That overhead, he said, is less of a concern than preemption causing ""material changes to a random subset of key benchmarks that specific enterprise customers care about"", so `PREEMPT_DYNAMIC` works well as it is. 

Zijlstra also said that, since `PREEMPT_DYNAMIC` seems to work for distributors, he is open to removing the other options. While the connection wasn't made in the conversation, doing so might solve the original problem as well. If the kernel is always maintaining the information needed to know when preemption is safe, that information can be used for a safe implementation of `TIF_ALLOW_RESCHED`. It may not come to that, though; the conversation is ongoing and some more significant changes to preemption are being considered; stay tuned for [the second part of this series](<https://lwn.net/Articles/945422/>) once the dust settles a bit.
