---
title: "Sub-schedulers for sched_ext"
url: https://lwn.net/Articles/1056014/
date: "January 29, 2026"
category: "BPF-CPU scheduling; Scheduler-Extensible scheduler class"
author: "By Jonathan Corbet January 29, 2026"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 29, 2026

The [extensible scheduler class](<https://lwn.net/Articles/922405/>) (sched_ext) allows the installation of a custom CPU scheduler built as a set of BPF programs. Its merging for the 6.12 kernel release moved the kernel away from the "one scheduler fits all" approach that had been taken until then; now any system can have its own scheduler optimized for its workloads. Within any given machine, though, it's still "one scheduler fits all"; only one scheduler can be loaded for the system as a whole. The [sched_ext sub-scheduler patch series](<https://lwn.net/ml/all/20260121231140.832332-1-tj@kernel.org>) from Tejun Heo aims to change that situation by allowing multiple CPU schedulers to run on a single system. 

Sched_ext was built around the idea that no scheduler can be optimized for every possible workload that it may encounter. The sub-scheduler work extends that idea by saying that no scheduler — even a sched_ext scheduler — can be prepared to obtain optimal performance from every workload that a given system may run. From the cover letter: 

> Applications often have domain-specific knowledge that generic schedulers cannot possess. Database systems understand query priorities and lock holder criticality. Virtual machine monitors can coordinate with guest schedulers and handle vCPU placement intelligently. Game engines know rendering deadlines and which threads are latency-critical. 

A system that runs a single workload can also run a special-purpose scheduler optimized for that workload. But the owners of systems tend to want to keep them busy, and that means running multiple workloads on the same machine. If two workloads on the same system would benefit from different scheduling algorithms, at least one of them is going to end up with sub-optimal performance. 

The solution is to allow sched_ext schedulers to be attached to control groups. Each task in the system will be governed by the scheduler attached to its control group (or to the nearest ancestor group). The kernel has long supported the [CPU controller](<https://docs.kernel.org/admin-guide/cgroup-v2.html#cpu>), which allows an administrator to allocate CPU resources across control groups. Interestingly, the sub-scheduler feature is not tied to the CPU controller; it is, instead, tied directly to the control-group mechanism. As a result, the CPU controller is still in charge of how much CPU time each group gets, while the sub-schedulers manage how that CPU time is used by the processes within their respective groups. 

Control groups with attached schedulers can be nested up to four levels deep. Any scheduler that is to be the parent of another in the control-group hierarchy must be written with that responsibility in mind. The attachment of a sub-scheduler to a control group will only succeed if the parent controller allows it. The parent scheduler also controls when the sub-schedulers' `dispatch()` callbacks are invoked. This callback instructs the scheduler to choose the next tasks to run and add them to a specific CPU's dispatch queue. So, in other words, the parent scheduler controls when a given workload (represented by a control group) can run, the sub-scheduler controls how the processes that make up that workload access the CPU, and the CPU controller is in charge of how much CPU time is available for them to run. 

The kernel exports a long list of kfuncs that allow sched_ext programs to operate on the scheduler and the processes running under it. Those kfuncs will need to be generalized so that, rather than operating on _the_ scheduler, they operate on the appropriate sub-scheduler. A system that is running multiple schedulers must also take extra care to ensure that these schedulers do not interfere with each other. For example, a BPF program that is running on behalf of a given scheduler should not be able to affect — or even see — any other schedulers that might be present. So the generalization of the sched_ext kfuncs must be carried out in a way that preserves the security and robustness of the system as a whole. 

To that end, many of those kfuncs have been augmented with an [implicit argument](<https://lwn.net/Articles/1055559/>) that gives them access to the [`bpf_prog_aux` structure](<https://elixir.bootlin.com/linux/v6.18.6/source/include/linux/bpf.h#L1602>) associated with the running task; from there, they can obtain a pointer to the sub-scheduler data they should be working with. The BPF programs themselves need never specify which scheduler they are operating on, and have no ability to operate on any scheduler other than the one they are attached to. The kernel is able to ensure that they are always tied to the correct sub-scheduler. 

Similarly, sched_ext programs must be prevented from operating on processes other than the ones running under the scheduler they implement. The kernel already maintains a structure ([`struct sched_ext_entity`](<https://elixir.bootlin.com/linux/v6.18.6/source/include/linux/sched/ext.h#L139>)) in the task structure that holds the information needed to manage each task with sched_ext. The new series adds a new field to that structure (called `sched`) pointing to the (sub-)scheduler that is in control of that task. Any kfunc that operates on a process can use this information to be sure that the process is, indeed, under the purview of the scheduler that is trying to make the change. 

Sched_ext is designed with the intent of keeping faulty schedulers from causing too much damage. When a problem is detected with a running scheduler (for example, a runnable task is not dispatched to a CPU within a reasonable time period), that scheduler is put into "bypass mode". This mode is also entered when a scheduler is being deliberately shut down. In bypass mode, the scheduler is deactivated, and all tasks running under it are placed under a simple FIFO scheduler. In current kernels, that bypass scheduler is global. 

In a system with multiple schedulers, though, allowing one sub-scheduler to toss processes into a global FIFO queue could lead to interference with other sub-schedulers. So, when a sub-schedulers are in use, the parent scheduler, if it exists, will inherit tasks from sub-schedulers that go into the bypass mode. In the other direction, if a parent scheduler goes into bypass mode, any schedulers below it in the hierarchy will also be placed in bypass mode. 

The current patch set (version 1, though there was [an RFC version](<https://lwn.net/ml/all/20250920005931.2753828-1-tj@kernel.org/>) in September 2025 as well) is not yet a complete implementation. It covers primarily the dispatch path — where a given scheduler sends a task to a CPU to execute. There are a couple of important phases that happen before dispatch, though, that have not been addressed in this series: 

  * The `select_cpu()` callback is invoked when a task first wakes up. The scheduler should decide what to do with the task, including selecting a CPU for it to run on (though the selection is not final at this stage). 
  * The `enqueue()` callback will actually put the task into a dispatch queue. That queue might be a specific CPU's local dispatch queue, but it may also be some other queue maintained by the scheduler, from which the task will be put into a CPU-local queue at a later time. 

These callback paths will clearly need to be worked out for a complete sub-scheduler implementation. What is there now, though, is enough to show how the whole thing is intended to work. There is [a modified version](<https://lwn.net/ml/all/20260121231140.832332-35-tj@kernel.org>) of the [scx_qmap scheduler](<https://github.com/torvalds/linux/blob/master/tools/sched_ext/scx_qmap.c>) that is able to operate as both a parent and a sub-scheduler; it shows, in a relatively simple form, the type of changes that are necessary to the schedulers themselves. 

As noted, this is early-stage work, and it is not yet complete. One should thus not expect to see sub-scheduler support in the kernel for some time yet. It is not hard to see how this feature could be useful on systems running a variety of workloads, though, so there will be a clear motivation to push it over the finish line. At that point, the "one scheduler fits all" model will have been left far behind.
