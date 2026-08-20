---
title: "KS2009: The state of the scheduler"
url: https://lwn.net/Articles/357471/
date: "October 19, 2009"
category: Scheduler
author: "By Jonathan Corbet October 19, 2009 LWN's 2009 Kernel Summit coverage"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 19, 2009

* * *

LWN's [2009 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2009/>)

Peter Zijlstra started out the "state of the scheduler" session by noting that, on occasion, Con Kolivas surfaces with a new scheduler and people start sending in scheduler bugs. He would really like to short out part of the process and get problem reports regardless of Con's release schedule. Scheduling is hard, with a lot of conflicting requirements. But, if people send in bug reports, preferably with reproducible test cases, the scheduler developers will do their best to fix things. 

Currently the most interesting work in the scheduling area is around [deadline scheduling](<http://lwn.net/Articles/356576/>). There are a number of workloads where static priorities just do not map well to the problem space. The biggest change in 2.6.32, instead, is a reworking of the load balancing code. Among other things, the load balancer is becoming more aware of "CPU capacity." Thus far, the scheduler has always assumed that each processor is capable of performing the same amount of work. There are a number of things - runtime power management, for example - which can invalidate that assumption. So the new load balancing code tries to observe what each CPU is accomplishing and come up with an estimate of capacities to be used in scheduling decisions. 

It was noted that there are still some problems associated with scheduling on NUMA systems, but they were not described in any detail. 

The discussion turned to scheduler benchmarks for desktop workloads. It was observed that kernel developers tend to optimize for kernel builds, which is not necessarily a representative workload for the wider user base. Peter noted that the perf tool looks like it will be able to help in this regard; users can use it to record system traces while running problematic workloads, then the developers can use the recorded data to reproduce the problem. Linus claimed that, for many desktop interactivity problems, the scheduler is irrelevant - the real problems tend to be at the I/O scheduler or filesystem levels. Others disagree, though, stating that some problems still exist within the CPU scheduler. 

Hyperthreading is coming back with newer CPUs; what is the scheduler doing to improve performance on hyperthreaded systems? The problem here is that a process running on a hyperthreaded CPU will adversely impact the performance of the sibling CPU. The variable capacity code should help here, but there were complaints that this code is limited by what has been observed in the past. The future can be different, especially if the workload shifts. There's only so much that can be done about that; predicting the future remains difficult. 

Can performance counters be used to better estimate CPU capacity? The answer appears to be negative: reading a performance counter is a very expensive operation. Trying to integrate performance counters into scheduling decisions would kill performance. 

The cost of the scheduler itself was raised as a problem, especially on embedded systems. It's not clear, though, how much of the problem is really the scheduler, and how much is other work being done on the timer tick. One observation was that indirect function calls (used within the scheduler to call into the specific scheduler classes) can play havoc with branch prediction on some architectures. Linus suggested that people encountering this problem should "get an x86 and quit whining," but chances are that solution is not good for everybody. It may make sense to turn the indirect function calls into some sort of switch statement, at least for some architectures. 

The discussion then shifted to the problem of certain proprietary databases which run specific threads under the realtime scheduling classes. They are apparently working around certain problems associated with their use of user-space spinlocks. It was agreed that this code should be using futexes rather than rolling their own locking schemes. 

But it is not that simple: Chris Mason observed that he had tried to get the database folks to use futexes, and they had tried it. The resulting performance was much worse, and he "lost the argument horribly." One problem is that futexes lack adaptive spin capability, which would make them perform better. But there were also a lot of complaints about the implementation of locking within glibc. 

For all of the usual reasons, nobody feels particularly optimistic about being able to get fixed locking into glibc. So there is talk of creating a separate user-space locking library as a way of routing around the problem and making reasonable locking available to applications. It might be a reimplementation of POSIX threads, or it could be a simpler library focused on locking primitives. Creating this library could be challenging, but there could be some nice payoffs as well. It might, for example, become possible to provide a lockdep-like debugging facility to user space. 

There's also a strong desire within the Samba community for a per-thread filesystem UID. The kernel can do this now, but glibc hides the capability so applications cannot use it. A separate threads/locking library could make that feature available to applications as well. 

[Next: The end-user panel](<https://lwn.net/Articles/357477/>)
