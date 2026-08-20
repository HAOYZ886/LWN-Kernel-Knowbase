---
title: The extensible scheduler class
url: https://lwn.net/Articles/922405/
date: "February 10, 2023"
category: "BPF-CPU scheduling; Releases-6.12; Scheduler-Extensible scheduler class"
author: "By Jonathan Corbet February 10, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 10, 2023

It was only a matter of time before somebody tried to bring BPF to the kernel's CPU scheduler. At the end of January, Tejun Heo posted [the second revision of a 30-part patch series](<https://lwn.net/ml/linux-kernel/20230128001639.3510083-1-tj@kernel.org/>), co-written with David Vernet, Josh Don, and Barret Rhoden, that does just that. There are clearly interesting things that could be done by deferring scheduling decisions to a BPF program, but it may take some work to sell this idea to the development community as a whole. 

The core idea behind BPF is that it allows programs to be loaded into the kernel from user space at run time; using BPF for scheduling has the potential to enable significantly different scheduling behavior than is seen in Linux systems now. The idea of "pluggable" schedulers is not new; it came up in [this 2004 discussion](<https://lwn.net/Articles/109458/>) of yet another doomed patch series from Con Kolivas, for example. At that time, the idea of pluggable schedulers was strongly rejected; only by focusing energy on a single scheduler, it was argued, could the development community find a way to satisfy all workloads without filling the kernel with a confusion of special-purpose schedulers. 

Of course, the idea that the kernel only has one CPU scheduler is not quite accurate; there are actually several of them, including the realtime and deadline schedulers, that applications can choose between. But almost all work on Linux systems runs under the default "completely fair scheduler", which indeed does a credible job of managing a wide variety of workloads on everything from embedded systems to supercomputers. There is always a desire for better performance, but there have been almost no requests for a pluggable scheduler mechanism for years. 

Why, then, is the BPF mechanism being proposed now? In clear anticipation of a long discussion, the cover letter for the patch series describes the motivation behind this work at great length. In short, the argument goes, the ability to write scheduling policies in BPF greatly lowers the difficulty of experimenting with new approaches to scheduling. Both our workloads and the systems they run on have become much more complex since the completely fair scheduler was introduced; experimentation is needed to develop scheduling algorithms that are suited to current systems. The BPF scheduling class allows that experimentation in a safe manner without even needing to reboot the test machine. BPF-written schedulers can also improve performance for niche workloads that may not be worth supporting in the mainline kernel and are much easier to deploy to a large fleet of systems. 

#### Scheduling with BPF

The patch set adds a new scheduling class, called `SCHED_EXT`, that can be selected with a [`sched_setscheduler()`](<https://man7.org/linux/man-pages/man2/sched_setscheduler.2.html>) call like most others (selecting `SCHED_DEADLINE` is a bit more complicated). It is an unprivileged class, meaning that any process can place itself into `SCHED_EXT`. `SCHED_EXT` is placed between the idle class (`SCHED_IDLE`) and the completely fair scheduler (`SCHED_NORMAL`) in the priority stack. As a result, no `SCHED_EXT` scheduler can take over the system in a way that would prevent, for example, an ordinary shell session running as `SCHED_NORMAL` from running. It also suggests that, on systems where `SCHED_EXT` is in use, the expectation is that the bulk of the workload will be running in that class. 

The BPF-written scheduler is global to the system as a whole; there is no provision for different groups of processes to load their own schedulers. If there is no BPF scheduler loaded, then any processes that have been put into the `SCHED_EXT` class will be run as if they were in `SCHED_NORMAL` instead. Once a BPF scheduler is loaded, though, it will take over the responsibility for all `SCHED_EXT` tasks. There is also a magic function that a BPF scheduler can call (`scx_bpf_switch_all()`) that will move _all_ processes running below realtime priority into `SCHED_EXT`. 

A BPF program implementing a scheduler will normally manage a set of dispatch queues, each of which may contain runnable tasks that are waiting for a CPU to execute on. By default, there is one dispatch queue for every CPU in the system, and one global queue. When a CPU is ready to run a new task, the scheduler will pull a task off of the relevant dispatch queue and give it the CPU. The BPF side of the scheduler is mostly implemented as a set of callbacks to be invoked via [an operations structure](<https://lwn.net/Articles/811631/>), each of which informs the BPF code of an event or a decision that needs to be made. The list is long; the full set can be found in [`include/sched/ext.h`](<https://github.com/htejun/sched_ext/blob/sched_ext-v2/include/linux/sched/ext.h#L165>) in the `SCHED_EXT` repository branch. This list includes: 

> `prep_enable()`
> `enable()`
>      The first callback informs the scheduler of a new task that is entering `SCHED_EXT`; the scheduler can use it to set up any associated data for that task. `prep_enable()` is allowed to block and can be used for memory allocations. `enable()`, which cannot block, actually enables scheduling for the new task. 
> `select_cpu()`
>      Select a CPU for a task that is just waking up; it should return the number of the CPU to place the task on. This decision can be revisited before the task actually runs, but it may be used by the scheduler to wake the selected CPU if it is currently idle. 
> `enqueue()`
>      Enqueue a task into the scheduler for running. Normally this callback will call `scx_bpf_dispatch()` to place the task into the chosen dispatch queue, from which it will eventually be run. Among other things, that call provides the length of the time slice that should be given to the task once it runs. If the slice is specified as `SCX_SLICE_INF`, the CPU will go into the [tickless mode](<https://lwn.net/Articles/549580/>) when this task runs. 
> 
> It's worth noting that `enqueue()` is not _required_ to put the task into any dispatch queue; it could squirrel that task away somewhere for the time being if the task should not run immediately. The kernel keeps track, though, to ensure that no task gets forgotten; if a task languishes for too long (30 seconds by default, though the timeout can be shortened), the BPF scheduler will eventually be unloaded. 
> `dispatch()`
>      Called when a CPU's dispatch queue is empty; it should dispatch tasks into that queue to keep the CPU busy. If the dispatch queue remains empty, the scheduler will try to grab tasks from the global queue instead. 
> `update_idle()`
>      This callback informs the scheduler when a CPU is entering or leaving the idle state. 
> `runnable()`
> `running()`
> `stopping()`
> `quiescent()`
>      These all inform the scheduler about status changes for a task; they are called when, respectively, a task becomes runnable, starts running on a CPU, is taken off a CPU, or becomes no longer runnable. 
> `cpu_acquire()`
> `cpu_release()`
>      Inform the scheduler about the status of the CPUs in the system. When a CPU becomes available for the BPF scheduler to manage, a callback to `cpu_acquire()` informs it of the fact. The loss of a CPU (because, perhaps, a realtime scheduling class has claimed it) is notified with a call to `cpu_release()`. 

There are numerous other callbacks for the management of control groups, CPU affinity, core scheduling, and more. There is also a set of functions that the scheduler can call to affect scheduling decisions; for example, `scx_bpf_kick_cpu()` can be used to preempt a task running on a given CPU and call back into the scheduler to pick a new task to run there. 

#### Examples

The end result is a framework that allows the implementation of a wide range of scheduling policies in BPF code. To prove the point, the patch series includes a number of sample schedulers. [This patch](<https://lwn.net/ml/linux-kernel/20230128001639.3510083-15-tj@kernel.org/>) contains a minimal "dummy" scheduler that uses the default for all of the callbacks; it also has a basic scheduler that implements five priority levels and shows how to stash tasks into BPF maps. ""While not very practical, this is useful as a simple example and will be used to demonstrate different features"". 

Beyond that, there is [a "central" scheduler](<https://lwn.net/ml/linux-kernel/20230128001639.3510083-20-tj@kernel.org/>) that dedicates one CPU to scheduling decisions, leaving all others free to run the workload. A [later patch](<https://lwn.net/ml/linux-kernel/20230128001639.3510083-23-tj@kernel.org/>) adds tickless support to that scheduler and concludes: 

> While scx_example_central itself is too barebone to be useful as a production scheduler, a more featureful central scheduler can be built using the same approach. Google's experience shows that such an approach can have significant benefits for certain applications such as VM hosting. 

As if that weren't enough, [`scx_example_pair`](<https://lwn.net/ml/linux-kernel/20230128001639.3510083-24-tj@kernel.org/>) implements a form of [core scheduling](<https://lwn.net/Articles/861251/>) using control groups. The [`scx_example_userland`](<https://lwn.net/ml/linux-kernel/20230128001639.3510083-30-tj@kernel.org/>) scheduler ""implements a fairly unsophisticated sorted-list vruntime scheduler in userland to demonstrate how most scheduling decisions can be delegated to userland"". The series concludes with the [Atropos scheduler](<https://lwn.net/ml/linux-kernel/20230128001639.3510083-31-tj@kernel.org/>), which features a significant user-space component written in Rust. The cover letter describes one more, [`scx_example_cgfifo`](<https://github.com/htejun/sched_ext/commit/f2fcd3147fb6286e0a35fcbed33c3bac69546a96>), which wasn't included because it depends on the still out-of-tree [BPF rbtree patches](<https://lwn.net/ml/bpf/20230131180016.3368305-1-davemarchevsky@fb.com/>). It ""provides FIFO policies for individual workloads, and a flattened hierarchical vtree for cgroups"", and evidently provides better performance than `SCHED_NORMAL` for an Apache web-serving benchmark. 

#### Prospects

This patch set is in its second posting and has, so far, not drawn a lot of review comments; perhaps it is too big to bikeshed. Scheduler maintainer Peter Zijlstra responded to [the first version](<https://lwn.net/ml/linux-kernel/20221130082313.3241517-1-tj@kernel.org/>), though, [saying](<https://lwn.net/ml/linux-kernel/Y5b2btWFJeEfTyJg@hirez.programming.kicks-ass.net/>): ""I hate all of this. Linus NAK'ed loadable schedulers a number of times in the past and this is just that again -- with the extra downside of the whole BPF thing on top"". He then proceeded to review many of the component patches, though, suggesting that he may not intend to reject this work outright. 

Even so, the BPF scheduler class will clearly be a large bite for the core kernel community to swallow. It adds over 10,000 lines of core code and exposes many scheduling details that have, thus far, been kept deep within the kernel. It would be an acknowledgment that one general-purpose scheduler cannot optimally serve all workloads; some may worry that it would mark an end to work on the completely fair scheduler toward that goal and an increase in fragmentation across Linux systems. The BPF-scheduling developers argue the opposite, that the ability to freely experiment with scheduling models would, instead, accelerate improvements to the completely fair scheduler. 

How this will play out is hard to predict, other than to note that the BPF juggernaut has, thus far, managed to overcome just about every objection that it has encountered. The days of locking up core-kernel functionality within the kernel itself seem to be coming to an end. It will be interesting to see what new scheduling approaches will be enabled by this subsystem.
