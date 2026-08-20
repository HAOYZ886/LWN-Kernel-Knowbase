---
title: "Scheduler medley: time-slice extension, sched_ext deadline servers, and LRU batching."
url: https://lwn.net/Articles/1029093/
date: "July 17, 2025"
category: "Scheduler-CPU isolation; Scheduler-Extensible scheduler class; Scheduler-Time-slice extension"
author: "By Jonathan Corbet July 17, 2025"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
July 17, 2025

Decades after its creation, the Linux CPU scheduler remains an area of active development; it is difficult to find a time slice to cover every interesting scheduler change. In an attempt to catch up, the time has come to round-robin through a few patches that have been circulating recently. The work at hand focuses on a new attempt at time-slice extension, the creation of a deadline server for sched_ext tasks, and keeping tasks on isolated CPUs from being surprised by LRU batching. 

#### Time-slice extension

The kernel is able to ensure that it will not be preempted whenever that would be especially inconvenient, for itself or for a task that it is running. User space, instead, has fewer options in that regard. As a result, it is possible for a user-space thread to lose the CPU while holding a lock that another thread needs, delaying the latter unnecessarily. Since lock-hold times are (or at least should be) short, delaying the preemption of a thread for a brief period while it holds a lock could improve the performance of the system overall. 

Earlier this year, Steve Rostedt [posted a patch series](<https://lwn.net/Articles/1009509/>) adding the capability for a thread to indicate that it is running in a critical section by way of a special field in the [`rseq` structure](<https://elixir.bootlin.com/linux/v6.14-rc2/source/include/uapi/linux/rseq.h#L62>) that is shared between the kernel and user space. In Rostedt's implementation, a counter was used, and any non-zero value stored there would indicate that a critical section was being executed. If the kernel encountered such a value at a time when it was planning to preempt that thread, it would, if possible, give that thread an additional 50µs of running time. The use of a counter was intended to support nested critical sections on the user-space side. Entry into a critical section would increment the counter, while exiting would decrement it; when the counter reached zero, the task would be known not to be running in a critical section. 

That patch set generated a fair amount of conversation and disagreement over how it should be implemented, with the end result that the work stalled and was never seriously considered for mainline inclusion. At the same time as Rostedt was working on his patch, though, Prakash Sangappa had been working on [a similar patch](<https://lwn.net/ml/all/20241113000126.967713-1-prakash.sangappa@oracle.com/>). Unlike Rostedt, Sangappa has continued with this work; [version 6 of his series](<https://lwn.net/ml/all/20250701003749.50525-1-prakash.sangappa@oracle.com>) was posted at the beginning of July. 

Sangappa initially did not use `struct rseq`, opting instead to create a separate memory region shared with user space. That has changed over the evolution of the patch; the current version uses a one-bit flag in that structure. The maximum deferral time in this series is 30µs, but there is also a sysctl knob that can be used to change that value or, by setting it to zero, to disable the feature entirely. Otherwise, on the surface at least, the patch is similar to Rostedt's version. 

Rostedt, though, had tied his mechanism to the relatively new [lazy preemption](<https://lwn.net/Articles/994322/>) scheduling mode; that decision had proved controversial. Sangappa's patch, instead, applies to tasks running in any scheduling mode. Thomas Gleixner [complained](<https://lwn.net/ml/all/87cyakmhdv.ffs@tglx/>) about that design, saying that it would allow a normal-priority task to delay a realtime task, which ""is fundamentally wrong"". Gleixner said that preemption deferral should only happen in situations where lazy preemption would apply — when there is not a high-priority task waiting for the CPU. 

Peter Zijlstra, though, [disagreed](<https://lwn.net/ml/all/20250701105653.GO1613376@noisy.programming.kicks-ass.net/>), saying that the potential delay for a realtime task is less than what could already happen when an ordinary system call is made. Additionally, he said, tying deferral to lazy preemption would prevent realtime threads from using it, but the feature might be just as useful for those threads. Gleixner [acquiesced](<https://lwn.net/ml/all/87wm8skrzj.ffs@tglx/>) to this point of view, but said that the feature needs a mechanism to enable or disable it on a per-process basis; Rostedt [agreed](<https://lwn.net/ml/all/20250701104914.7bb80161@batman.local.home/>) with that request. 

Nobody seems to disagree with the fundamental objective of this patch series, though, so it is mostly a matter of getting agreement on the implementation. It is hard to say how many more rounds will be required to reach that point, but eventually it seems that this feature should find its place in the mainline kernel. 

#### Deadline servers for sched_ext

The [extensible scheduler class](<https://lwn.net/Articles/922405/>) (sched_ext) allows custom CPU schedulers to be loaded from user space as a set of BPF programs. Those custom schedulers can implement whatever special algorithm is needed to ensure that a given workload runs efficiently, often optimizing the system in ways that the in-kernel scheduler is unable to achieve. If, however, a realtime task runs for 100% of the CPU time, sched_ext tasks will be unable to run and all that work will have gone for nothing; the problem gets even worse if one of the blocked sched_ext tasks represents the system administrator trying to regain control. 

The same problem applies to tasks running in the `SCHED_NORMAL` class, of course, and various attempts have been made to solve it over the years. The current solution is [deadline servers](<https://lwn.net/Articles/934415/>), special kernel processes that run in the deadline-scheduling class, which outranks even the normal realtime classes. A deadline-server thread is configured to run for a maximum of 5% of the available CPU time; it uses that time to run `SCHED_NORMAL` tasks in the usual way. This mechanism ensures that a minimum of CPU time remains available for non-realtime tasks, but allows realtime tasks to use that time if there are no other requests for it. 

The solution, as [implemented by Joel Fernandes](<https://lwn.net/ml/all/20250702232944.3221001-1-joelagnelf@nvidia.com>), is to add a deadline server for sched_ext tasks too, ensuring that they are not crowded out of the CPU entirely. Given that the infrastructure for deadline servers exists at this point, the amount of work required to effect this change is relatively small; still, six versions of the series have been posted so far. That said, comments on the work seem to be winding down, so it may be getting close to ready. 

#### LRU batching on isolated CPUs

There are some workloads that _really_ do not want to be interrupted; they need 100% of the time on the CPU(s) where they have been placed. The kernel supports those applications with its CPU-isolation feature. When a CPU is isolated, all of the normal kernel housekeeping work, including interrupt handling, is moved to other CPUs. The scheduler tick is disabled, and the designated task begins running there. As long as that task does not make a system call, it should not have to compete with any other work for that CPU. 

CPU isolation is rather poorly documented within the kernel; there is information to be found in [this 2020 LWN article](<https://lwn.net/Articles/816298/>) and [this article series](<https://www.suse.com/c/cpu-isolation-introduction-part-1/>) on the SUSE blog. 

Memory-management tasks, such as working through folios on the least-recently-used (LRU) lists, qualifies as the kind of work that should not be done on isolated CPUs. But, as Frederic Weisbecker points out in [this patch series](<https://lwn.net/ml/all/20250703140717.25703-1-frederic@kernel.org>), the isolation in current kernels is not yet perfect. Specifically, the memory-management subsystem maintains a set of per-CPU arrays of folios that have been marked for LRU processing. Folios can be added to those arrays quickly, then processed at leisure when the kernel concludes that a good time has been reached. 

In current kernels, though, one CPU can, at times, make that decision for another. If the memory-management subsystem notices that there are folios in another CPU's arrays awaiting attention, it can remotely trigger that processing, even if the target CPU is running in the isolated mode. That will result in the workload losing access to the CPU while the LRU arrays are drained, breaking the promise that had been made. 

Normally, if the workload is running in user space, one would not expect folios to end up in the per-CPU arrays to begin with. There can be problems with pages left over when the CPU enters the isolated mode, though. Additionally, even the most user-space-intensive task may need to make a system call every now and then, and those can end up queuing more folios for processing. 

Weisbecker's solution is to stop the remote triggering of LRU processing on isolated CPUs. Instead, a per-CPU flag will be set, and the target CPU will perform this processing on return from the next system call. That will still delay the isolated workload, but that delay will happen when the task makes a system call and delays are already expected. That should allow this work to be done at a time when it will not create problems for the isolated workload. 

There have been some comments on the implementation that suggest the need for a least one more revision of this series. Once this work goes in, though, the kernel will have the basic infrastructure for the deferral of work on isolated CPUs; it would not be surprising to see other kernel housekeeping work end up being moved into that structure as well.
