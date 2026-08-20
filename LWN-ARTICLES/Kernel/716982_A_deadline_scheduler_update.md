---
title: A deadline scheduler update
url: https://lwn.net/Articles/716982/
date: "March 14, 2017"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "By Jonathan Corbet March 14, 2017 Linaro Connect"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 14, 2017

* * *

[Linaro Connect](<https://lwn.net/Archives/ConferenceByYear/#2017-Linaro_Connect>)

The deadline CPU scheduler has come a long way, Juri Lelli said in his 2017 Linaro Connect session, but there is still quite a bit of work to be done. While this scheduler was originally intended for realtime workloads, there is reason to believe that it is well suited for other settings, including the embedded and mobile world. In this talk, he gave a summary of what the deadline scheduler provides now and the changes that are envisioned for the near (and not-so-near) future. 

The deadline scheduler was merged in the 3.14 development cycle. It adds a realtime scheduling policy that, in many ways, is more powerful than traditional, priority-based scheduling. It allows for the specification of explicit latency constraints and avoids starvation of processes by design. The scheduler has better information about the constraints of the workload it is running and can thus make better decisions. 

The kernel's scheduler is based on the earliest deadline first (EDF) algorithm, under which the process with the first-expiring deadline is the one that is chosen to run. EDF is enhanced with the constant bandwidth server (CBS) algorithm (described in detail in [this article](<https://lwn.net/Articles/398470/>)), which prevents a process that is unable to run for much of its period from interfering with others. Essentially, CBS says that a deadline process must use its CPU-time reservation over the course of its scheduling period, rather than procrastinating and expecting the full reservation to be available right before the deadline. The result is a scheduler that provides strong temporal isolation for tasks, where no process can prevent another from satisfying its deadlines. 

At the moment, the mobile and embedded development community is putting a lot of effort into energy-aware scheduling. This work has a valuable goal — making scheduling decisions that minimize a system's energy use — but it has proved to be hard to get upstream, though it has been merged into the Android common kernel. For many workloads, it may be that deadline scheduling is a better fit for energy-conscious workloads in the end, Lelli said 

A new feature under development is bandwidth reclaiming. One of the core features of deadline scheduling is that, when a process exceeds its CPU-time reservation (its CPU "bandwidth"), the scheduler will simply throttle it until its next scheduling period. This throttling is needed to ensure that the process cannot interfere with other processes on the [![\[Juri Lelli\]](https://static.lwn.net/images/conf/2017/connect/JuriLelli-sm.jpg)](<https://lwn.net/Articles/716983/>) system, but it can be a problem if a process occasionally has a legitimate need for more than its allotted time. Bandwidth reclaiming just does the obvious thing: it gives that process more time if it's not needed by other processes in the system. 

What is perhaps less obvious is the determination of how much CPU time is not actually needed. This is done with the GRUB ("greedy utilization of unused bandwidth") algorithm described in [this paper [PDF]](<http://disi.unitn.it/~abeni/reclaiming/rtlws14-grub.pdf>). In short, GRUB tracks how much of the available CPU time is actually being used by the set of running deadline tasks and, from that, forms an estimate of how much CPU time will go unused. With that estimate in place, handing out some of the spare time to a deadline process that finds itself in need is a relatively straightforward business. 

CPU-frequency scaling is an important tool in the power-management portfolio, but it has traditionally not played well with realtime scheduling algorithms, including deadline scheduling. In current kernels, it is assumed that realtime tasks need the full power of the CPU, so the presence of such tasks will cause the CPU to run at full speed. That may be wasteful, though, if the deadline processes running do not need that much CPU time. 

Fixing that problem requires a number of changes, starting with the fact that the deadline scheduler itself assumes that the CPU will be running at full speed. The scheduler needs to be fixed so that it can scale reservation times to match the current speed of the processor. This awareness needs to be expanded to heterogeneous multiprocessing systems (such as big.LITTLE, where not all processors in the system are equally fast) as well. 

Once that is done, it would be beneficial to be able to drive CPU-frequency selection from the deadline scheduler as well. The CFS scheduler used for normal tasks uses the [per-entity load tracking](<https://lwn.net/Articles/531853/>) mechanism to help with frequency selection, but the deadline scheduler currently just pushes the frequency to the maximum. Once the bandwidth reclaiming code is in, it will become possible to measure and use the actual load added by deadline tasks. At that point, a CPU frequency that efficiently gets all of the necessary work done can be chosen. 

Of course, there are always a number of fiddly details to take care of. For example, on ARM systems, CPU-frequency changes are done in a separate worker thread. For CPU scaling and deadline scheduling to work together, a way for that thread to preempt deadline tasks (which are normally not preemptable) will need to be found. 

The deadline scheduler currently works at the level of individual processes; it does not work with control groups. But there are times when it might make sense to give a deadline reservation to a group of processes. Lelli cited virtual-machine threads and rendering pipelines as a couple of candidate workloads for group deadline scheduling. The implementation envisioned here would be a sort of two-level hybrid hierarchy. At the top level, the EDF algorithm would be used to pick the next group to execute; within that group, though, normal realtime scheduling (FIFO or round-robin) would be used instead. Once this feature works, he said, it could supplant the longstanding [realtime throttling](<https://lwn.net/Articles/296419/>) mechanism. 

Looking further ahead, Lelli said there is a scheme to extend the bandwidth reclaiming mechanism to allow priority demotion. Once a process exceeds its reservation, it will continue to run, but as a normal process without realtime priority. That priority will be restored once the next scheduling period starts. There is also a strong desire to have fully energy-aware scheduling in the deadline scheduler. 

A more distant wishlist item is support for single-CPU affinity. The priority inheritance mechanism could also stand some improvements. Currently, a task that blocks a deadline task will inherit that task's deadline. Replacing that with an algorithm like the [multiprocessor bandwidth inheritance protocol [PDF]](<https://www.iris.sssup.it/retrieve/handle/11382/305127/1017/paper.pdf>) would be desirable. There is also a wish for a dynamic feedback mechanism that could adjust a process's reservation based on its observed needs. But, for the time being, he said, nobody is actually working on these items. 

The [video of this session](<http://connect.linaro.org/resource/bud17/bud17-307/>) is available. 

[Thanks to Linaro and the Linux Foundation for funding your editor's travel to Connect.]
