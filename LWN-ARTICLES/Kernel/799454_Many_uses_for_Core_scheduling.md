---
title: Many uses for Core scheduling
url: https://lwn.net/Articles/799454/
date: "September 20, 2019"
category: "Scheduler-Core scheduling"
author: "By Jonathan Corbet September 20, 2019 LPC"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 20, 2019

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2019-Linux_Plumbers_Conference>)

Some new kernel features are welcomed by the kernel development community, while others are a rather harder sell. It is fair to say that [core scheduling](<https://lwn.net/Articles/780703/>), which makes CPU scheduling harder by placing constraints on which processes may run simultaneously in a core, is of the latter variety. Core scheduling was the topic of (at least) three different sessions at the 2019 Linux Plumbers Conference. One of the most interesting outcomes, perhaps, is that there are use cases for this feature beyond protection from side-channel attacks. 

#### Current status

The discussion started in a refereed-track talk by Julien Desfossez and Vineeth Remanan Pillai. Desfossez began by noting that core scheduling has been under development for about a year; its primary purpose is to make simultaneous multi-threading (SMT, or "hyperthreading") secure in the face of hardware vulnerabilities such as speculative-execution attacks. An SMT core contains two or more CPUs (sometimes called "hardware threads") that share a great deal of low-level hardware. That sharing, which includes a number of caches, makes SMT particularly vulnerable to cache-based side-channel attacks. For sites that are worried about such attacks, the only practical alternative now is to turn SMT off, which can have a significant performance impact for some workloads. Core scheduling was developed as a less drastic way to keep tasks that don't trust each other from executing in the same core at the same time. 

Pillai took over at this point, saying that core scheduling groups trusted tasks on a core. It treats the CPUs in an SMT core as a unit, finding the highest-priority task on all sibling CPUs; that task will drive the [![\[Julien Desfossez,
Vineeth Remanan Pillai\]](https://static.lwn.net/images/conf/2019/lpc/Defossez-Pillai-sm.jpg)](<https://lwn.net/Articles/799490/>) scheduling decisions. If another task can be found that is compatible with the high-priority task, it will be able to run on a sibling CPU; otherwise, that sibling will have to be forced idle. 

The first implementation of core scheduling was specific to KVM; it would only allow virtual CPU threads from the same virtual machine to share a core. It was then generalized, with the idea that administrators should be able to define the policy that sets the trust boundaries. The initial prototype uses control groups; an administrator can mark tasks as compatible by putting them in the same group, or setting the same value in the `cpu.tag` variable in multiple groups. The third version of the patch set is under discussion now; it is focused mostly on bug fixes and performance issues. 

There are indeed a few of these issues. The scheduler's `vruntime` value is used to compare tasks when making scheduling decisions; that works on a single CPU, but it was never intended to be used for cross-CPU comparisons. That can lead to starvation issues for some tasks. Current thinking is to treat these comparisons as more of a load-balancing issue, perhaps along with the creation of a normalized or core-wide `vruntime` value that will support more accurate comparisons. 

The forced idling of sibling CPUs is a necessary evil in core scheduling, but it's apparently even more evil than it really needs to be. In particular, a CPU-intensive process can cause a sibling to stay idle for a long time, starving the process executing there of CPU time. Somehow, the scheduler must learn to account for the forced-idle time to be able to trigger decisions (and switch to a starved task) at the right time. An alternative might be to create a special version of the idle task to run on a forced-idle CPU that can poke the scheduler when it is time to make a change. 

Desfossez returned to talk a bit about testing, which has been done extensively for this patch set. Performance-oriented patches always need to demonstrate a clear benefit; in this case, it's far from clear that core scheduling is always better than just turning off SMT. The testing infrastructure also uses tracing to verify correctness, ensuring that no incompatible tasks ever run together. 

Testing has revealed the fairness issues described above, and has shown that turning off SMT is indeed better for some workloads. In particular, I/O-intensive workloads do not benefit from core scheduling; he mentioned a MySQL benchmark that performs worse with it turned on. 

Future work includes a rethinking of how process selection works. There is also the little problem that, while core scheduling protects processes from each other, it does not protect the kernel against user space. Fixing that would require adding synchronization points on system calls and exit from virtual machines, which is likely to be expensive. There is probably no other way to protect the kernel from [MDS](<https://www.kernel.org/doc/html/latest/x86/mds.html#mds>) attacks, though. Finally, he said, the interface for identifying processes needs to be rethought. 

After the talk, Len Brown asked about how well the code would work on systems that have more than two siblings in an SMT core. Such systems do not exist now, but one can imagine CPU designers are thinking about such things. The answer was that the code is generic and should be able to handle that case, but it is hard to know for sure since there's no hardware available to test it on. 

#### Other uses

During the Scheduling Microconference, core scheduling was the topic of another set of sessions that were less focused on implementation details and more on other ways in which the feature might be used. Subhra Mazumdar started by describing a database use case from Oracle, which has its own virtualization setup and would like to use core scheduling to spread tasks optimally. But using core scheduling now leads to a significant (17-30%) performance decline, mostly as a result of the forced idling of CPUs. Often, a CPU goes idle when it could be running a task from another core elsewhere in the system. Mazumdar suggested that the scheduler's wakeup path needs to be changed to allow it to find a task with a matching tag anywhere in the system. 

In the discussion, it was repeated that core scheduling is unlikely to ever be better for all workloads. There were references, in particular, to [these benchmarks run by Mel Gorman](<https://lwn.net/ml/linux-kernel/20190425144619.GX18914@techsingularity.net/>), showing that enabling SMT can result in worse performance even in the absence of core scheduling. 

[![\[Aubrey Li\]](https://static.lwn.net/images/conf/2019/lpc/AubreyLi-sm.jpg)](<https://lwn.net/Articles/799512/>) Aubrey Li got up to discuss a different sort of use case for core scheduling: deep-learning training workloads. This kind of workload tends to use a lot of [AVX-512 instructions](<https://en.wikipedia.org/wiki/AVX-512>), which can give significant performance benefits. But these instructions, as it turns out, can reduce the maximum CPU frequency for the entire core; if an unrelated task is running elsewhere on the same core, it may be adversely affected by the AVX-512 use. Having two AVX-512-using processes on the same core, though, is no worse than having one there. 

It thus makes sense to keep processes making heavy use of those instructions together on the same core. Core scheduling can do this; his workload gets a 10% improvement in throughput and a 30% reduction in latency with it enabled. He thus believes that there would be value in merging core scheduling into the mainline. 

Jan Schönherr, instead, has a different sort of use case: isolating some processes, while forcing others to run together. His patch set, confusingly (in this context) named [coscheduling](<https://lwn.net/Articles/764482/>), allows an administrator to set policies that will force related processes to run on the same core while excluding others. The result should be the security benefits of core scheduling, but also some performance benefits from having related tasks share CPU resources. 

He was asked whether the existing cpuset functionality could handle this use case. The answer was that it works well, but only until the system is overcommitted. Once the load gets too heavy everything breaks down, and the simultaneous-execution property is lost. 

#### Realtime

One day later, during the Realtime Microconference, Peter Zijlstra led yet another session on core scheduling which, he said, is "all the rage these days". So many people want it that he's afraid there will be no alternative to merging it, even though it is not a complete solution to the side-channel problem. 

It turns out that realtime developers have come to find the idea attractive as well. The problem, from the realtime point of view, is that SMT is not deterministic, or as Zijlstra put it, it's "deterministically awful". Realtime users tend to disable it to avoid the latency problems [![\[Peter Zijlstra\]](https://static.lwn.net/images/conf/2019/lpc/PeterZijlstra-sm.jpg)](<https://lwn.net/Articles/799513/>) that it creates. But core scheduling can force sibling CPUs to go idle when a realtime process is running on the core, thus preventing this kind of interference. That opens the door to enabling SMT whenever a core has no realtime work, but effectively disabling it when realtime constraints apply, getting the best of both worlds. 

Using core scheduling this way raises some interesting questions, though; the one that was discussed during this session was the impact on the admission control enforced by the deadline scheduler. Admission control prevents the scheduler from accepting a deadline task if the CPU resources are not available to let that task meet its deadlines. Forcing CPUs idle affects the total amount of CPU time available; if admission control does not take that into account, the system may take on more work than it can handle. 

One possible solution that was raised in the session is to multiply a deadline process's worst-case execution time (essentially the amount of time it is allowed to run) by the number of CPUs in a core, since that process will, in effect, occupy all of those CPUs while it runs. There are a number of details to deal with, though, such as how to set the tag on such a task; allowing it to be done in a control group or with `prctl()` will be too late, potentially after the admission-control decision has been made. Perhaps `[sched_setattr()](<http://man7.org/linux/man-pages/man2/sched_setattr.2.html>)` could be enhanced for this purpose, but that would create two different ways to tag tasks for core scheduling. Zijlstra said that the developers would have to find an interface that works for all of the use cases. 

#### Getting it merged

Back in the Scheduler Microconference, Pillai wrapped up the session by stating the core scheduling is a big win for some use cases, and that it should be in the mainline kernel for those who can benefit from it. The feature will have to be turned off by default, though, since it is not beneficial to everybody. There is still the little problem that core scheduling does not protect the kernel; Pillai asserted that adding a security boundary on exit from virtual machines would be sufficient there. Providing isolation at system calls and interrupts is not as important. Thomas Gleixner disagreed strongly with that claim, though, saying that entry into the kernel is the same regardless of the mechanism used. 

Paul Turner said that protection against hardware vulnerabilities is not just a scheduling problem, and that core scheduling is insufficient regardless. Coscheduling will also prove necessary, he said, and probably something like the ill-fated (so far) [address-space isolation patches](<https://lwn.net/ml/linux-kernel/1562855138-19507-1-git-send-email-alexandre.chartre@oracle.com/>) as well. All of the pieces have to be looked at, and developers need to find a way to assemble them all. Gleixner agreed, but said that there also needs to be an understanding of the picture as a whole or the pieces will never fit. 

[Your editor thanks the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event.]
