---
title: Completing and merging core scheduling
url: https://lwn.net/Articles/820321/
date: "May 13, 2020"
category: "Scheduler-Core scheduling"
author: "By Jonathan Corbet May 13, 2020 OSPM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 13, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

[Core scheduling](<https://lwn.net/Articles/780703/>) is a proposed modification to the kernel's CPU scheduler that allows system administrators to control which processes can be running simultaneously on the same processor core. It was originally proposed as a security mechanism, but [other use cases](<https://lwn.net/Articles/799454/>) have shown up over time as well. At the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), a group of some 50 developers gathered online to discuss the current state of the core-scheduling patches and what is needed to get them into the mainline kernel. 

#### Status update

Vineeth Pillai started off by noting that, while mitigations have been developed for the known Spectre vulnerabilities, they do not provide complete protection on systems where simultaneous multi-threading (SMT or "hyperthreading") is in use. SMT creates the illusion of multiple CPUs running on shared hardware, increasing performance. The sharing of the underlying processor, however, provides numerous opportunities for speculative-execution vulnerabilities and the creation of covert channels. The only way to truly protect a system against these vulnerabilities is to disable SMT, which comes at a high performance cost for some workloads. 

Many Spectre vulnerabilities can be mitigated through selective cache flushing, Pillai said, but that is of limited utility when SMT is in use. Performing a cache flush will remove any information placed there before the flush, making it inaccessible afterward. To be effective, though, tasks that do not trust each other must not share the core between flushes. Without SMT, the kernel can perform a flush whenever it switches tasks, ensuring that this never happens. On an SMT core, though, each CPU schedules independently, so this kind of flushing discipline cannot be maintained. 

The way to avoid this problem is to keep tasks that don't trust each other from running on the same core — which means disabling SMT on current kernels. The alternative is core scheduling, where tasks in a given trust domain can be explicitly grouped together. The scheduler maintains [![\[Vineeth Pillai, Dario
Faggioli, and Tim Chen\]](https://static.lwn.net/images/conf/2020/ospm/coresched-sm.png)](<https://lwn.net/Articles/820334/>) core-wide knowledge of these trust domains and ensures that only tasks in the same domain run simultaneously on any given core. If there is no trusted companion for the highest-priority task, sibling CPUs can be forced idle while that task runs rather than run an untrusted task there. 

One open area, Pillai said, was in the area of load balancing, which doesn't currently work well with core scheduling. This could perhaps be improved by selecting a single run queue to hold the shared information needed for core scheduling. When a scheduling event happens, the highest-priority task would be chosen as usual. Then any sibling processors can be populated with matching tasks from across the system, should any exist. 

Core scheduling currently uses CPU control groups for grouping; there is a `cpu.tag` field that can be set to assign a "cookie" identifying the scheduling group a task belongs to. This was done for a quick and easy implementation, he said, and need not be how things will work in the end. There is a [red-black tree](<https://lwn.net/Articles/184495/>) in each run queue, ordered by cookie value, that is used to select tasks for sibling processors. 

The patch series is up to [version 5](<https://lwn.net/Articles/813808/>), which includes some load-balancing improvements. Earlier versions did not understand load balancing at all, so if a task was migrated to a CPU running (incompatible) tagged tasks, it could end up being starved for CPU time. A sixth revision is coming soon, he said. 

One challenge that has to be dealt with is comparing the priority of tasks across siblings. Within a run queue, a task's `vruntime` value is used to determine whether it should run next. This value is a sort of virtual run time, indicating how much CPU time the task has received relative to others (though it is scaled by the process priority and adjusted in various other ways), but this value is specific to each run queue. A `vruntime` in one run queue cannot be directly compared to a `vruntime` in another queue. 

One possible solution is to normalize these values. Each queue maintains a `min_vruntime` value, which is the lowest `vruntime` of any task in that queue. If a specific task's `vruntime` is normalized by subtracting the local `min_vruntime`, it can then be compared to a value in another run queue by adding that queue's `min_vruntime`. A solution based on this turned out to have starvation issues, though, leading to the creation of a core-wide `vruntime` instead; unfortunately, there are still starvation problems with this implementation, and discussions are ongoing. 

Work for the sixth revision includes some attention to the process selection code, which currently only picks the highest-priority task to run. That can cause starvation issues as well. There are also problems with tasks that go into the kernel (to handle an interrupt, for example) then end up being co-scheduled with an untrusted task; this will be expensive to fix. Some thought is going into the API, and perhaps switching to `prctl()` to set a task's grouping. 

Pillai concluded by asking whether this work should be merged into the mainline kernel. There are a number of arguments for that. Core scheduling shows better performance than just disabling SMT for a number of production use cases. It is controlled by a configuration option and is off by default even when configured in; there is no impact on performance when core scheduling is disabled. On the other hand, it's still only a partial mitigation for the problem, it has some fairness issues, there is code cleanup needed, and it still lacks a widely accepted API. 

#### IRQ leak mitigation and accounting

Joel Fernandes then took over to talk about one of the remaining Spectre mitigation issues: interrupt leaks. Google (his employer) wants to use core scheduling with Chrome OS, since there are some tangible benefits. In tests with the camera running on a Chromebook, core scheduling reduced key-press latency considerably while improving the camera's frame rate by 3%. But developers there are concerned that interrupts (of both the hard and soft variety) can cause untrusted tasks to run simultaneously with the kernel's interrupt handlers, exposing the kernel to data leaks. Core scheduling currently only works at the task level, with no control over interrupt handling, so it cannot address this problem. 

The proposed solution is the just-posted [IRQ leak mitigation patch](<https://lwn.net/ml/linux-kernel/20200510234652.249917-1-joel@joelfernandes.org/>). It works by tracking whenever a CPU within a core calls `irq_enter()` — when it starts to handle an interrupt, in other words. The first such call within a core will, if another CPU is running an untrusted task, cause an inter-processor interrupt forcing the other CPU to go idle. The scheduler itself also must be modified so that, when it switches from one task to another, it checks to see if another CPU is handling interrupts at the moment. If so it will wait until the coast is clear and caches have been flushed. 

There were some questions about how Chrome OS uses core scheduling. It seems that all "system tasks" are allowed to run together, outside of any core-scheduling group. Browser-based tasks (which are nearly all user tasks in Chrome OS) are each put into their own group, and thus run isolated. In other words, the _untrusted_ tasks are specially marked by the system. Peter Zijlstra remarked that this means tasks default to the trusted state, which seems insecure; he suggested that the default be untrusted instead. 

Juri Lelli asked about other scheduling classes; what happens if there is a realtime FIFO task in the system? Zijlstra answered that the usual ordering will be followed; the FIFO task, since it has the highest priority, will be picked first. Non-realtime tasks in the same group can then run on sibling processors, if they exist, though that would be a bit unusual since such tasks could interfere with the realtime task. 

Dario Faggioli talked for a bit about SUSE's use case for core scheduling: making sure that accounting for virtualized guests is accurate. A typical host system is running a lot of tasks, many of which represent the virtual CPUs (vCPUs) of different virtual machines. The scheduler can mix up those vCPUs in any way it likes, regardless of how they correspond to the virtual machines they emulate. 

Since tasks running on sibling CPUs are contending for the underlying hardware, they can affect each other's performance. Two vCPUs may appear to spend the same amount of time running on sibling CPUs, but one of those two may have actually consumed much more run time than the other. The result is unfair accounting of CPU time. Core scheduling can help by ensuring that vCPUs from the same virtual machine run on the same core, so that they only contend against each other; that makes the accounting more fair. 

Things can be improved further by defining the virtual machines so that some of their vCPUs are set up as SMT siblings, allowing the guest operating system to optimize its scheduling accordingly. That only works, though, if the virtual machine's description bears some relationship to the physical reality. Once again, core scheduling can make that happen. 

The security use case also applies to virtualization, Faggioli said. Core scheduling helps there, but does not yet appear to be a complete solution; the interrupt situation discussed by Fernandes is one place where work still needs to be done. A full solution is likely to need technologies like [address-space isolation](<https://lwn.net/Articles/803823/>) as well. 

#### Performance

Tim Chen presented a number of performance benchmarks that emulate several different use cases. A set of virtualization tests showed the system running at 96% of the performance of an unmodified kernel with core scheduling enabled; the 4% performance hit hurts, but it's far better than the 87% performance result measured for this workload with SMT turned off entirely. Some tests with the sysbench benchmark gave similar results for core scheduling, but turning off SMT cut performance nearly in half. The all-important kernel-build benchmark showed almost no penalty with core scheduling, while turning off SMT cost 8%. 

Julien Desfossez presented results from a MySQL benchmark showing performance dropping by 60% when core scheduling is used. Painful, but turning off SMT is far worse. A CPU-intensive benchmark based on Linpack showed core-scheduling performance that was slightly better than mainline, while turning off SMT incurs a 90% performance hit. 

Faggioli ran a set of tests on a 256-CPU AMD system, which does not need core scheduling for Spectre mitigation at all. He was interested in the performance cost of having core scheduling built into the kernel but turned off. Kernbench ran at 98.6% of mainline with 128 jobs, up to 99.9% with 256 jobs. Various other tests yielded similar numbers. 

There was some vague discussion of fairness testing — ensuring that all tasks get equal amounts of CPU time. The results were described as "messy" and hard to interpret, but the overall impression is that core scheduling yields less fair results overall. 

#### Discussion

The final part of this three-hour session was given over to unstructured discussion. Zijlstra, who will probably make the decision over whether core scheduling will be merged or not, started by saying that he would like to see some better documentation. In particular, he wants clear information on where core scheduling helps and where it doesn't; in which situations will it be helpful to turn core scheduling on? There are some things to work out, he said, including fairness and some problems with CPU hotplug. That can be sorted out, but the documentation is necessary to be able to move forward with this work. 

Dhaval Giani said that there are some cases that just don't work; not all problems are amenable to solution in the scheduler. Address-space isolation may also be needed to have a complete solution to the (known) Spectre problems. Zijlstra repeated that documentation covering what does work is needed. Then users can decide whether they care enough to turn it on for their specific situations. 

Aaron Lu said that there are still problems around `vruntime`. If two tasks have the same tag but differing weights (priorities), the core `vruntime` will become that of the lower-weight task since that task will not run as much. The difference between the two can become large. Zijlstra answered that unbounded divergence of `vruntime` between tasks is not a good thing, but renormalization is expensive. It is also unneeded. Once a sibling CPU has been idled, there is only one run queue that matters; that would be a good time to synchronize `vruntime` values. Lu expressed a desire to see a patch implementing that; Zijlstra expressed a weary willingness to try to find time to create one. 

Vincent Guittot raised the load-balancing issue; the results can be unfair, he said. If there are five tasks on a four-CPU system, one of those tasks will end up running slower than the others. He will be talking more about this issue later in the conference. In any case, Zijlstra said, the `vruntime` issues need to be worked out before load balancing can be resolved. 

As the session wound down, Giani tried to put together a list of the things that need to be worked out; these included `vruntime`, CPU hotplug, and the fairness issues. Pillai added starvation of untagged tasks to the list. Zijlstra asked if that problem was for tasks with a `cpu.tag` value of zero, which means no tag at all; Pillai said yes, but the special zero tag means that the task does not go into the core-scheduling red-black tree. Zijlstra suggested adding those tasks to the tree, which would remove the exceptional case and make things work again. 

Fernandes raised a related issue: that red-black tree contains task `vruntime` values that are used when selecting compatible tasks, but those values are not updated as the tasks run. Pillai said that this is a problem, old `vruntime` values can cause the scheduler to select the wrong task to run. Zijlstra said selecting a task for execution should remove it from this tree, as is done for the run-queue red-black tree. Doing this would slow things down, but it may be worth it; the penalty should be relatively small for virtualization workloads, since vCPUs are not rescheduled that often. 

The session ended with Zijlstra saying that this work looks ready to proceed, and that the remaining issues can be worked out on the mailing list.
