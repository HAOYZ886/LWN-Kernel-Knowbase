---
title: "Reports from OSPM 2026, day one"
url: https://lwn.net/Articles/1077759/
date: "June 22, 2026"
category: "Scheduler-Extensible scheduler class; Scheduler-and power management"
author: "By Jonathan Corbet June 22, 2026 OSPM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
June 22, 2026

* * *

[OSPM](<https://lwn.net/Archives/ConferenceIndex/#OS-Directed_Power-Management_Summit-2026>)

The [Power Management and Scheduling in the Linux Kernel Summit](<https://retis.santannapisa.it/ospm-summit/>), which still goes by the historical acronym OSPM, was held in Cambridge, UK, in mid-April. As has become traditional, the presenters at that event have since written summaries of their sessions, and this work has kindly been made available to LWN for publication. The first day's sessions covered a wide range of topics, including idle-state selection, user-space schedulers with sched_ext, lock-holder preemption, and much more. 

(See also: coverage from [day 2](<https://lwn.net/Articles/1078696/>) and [day 3](<https://lwn.net/Articles/1078697/>).) 

#### Success criteria for CPU idle state selection — Rafael J. Wysocki

In the first session of the conference, cpuidle subsystem maintainer Rafael Wysocki discussed issues related to measuring the quality of decisions made by cpuidle governors. 

He started by recapitulating the design and purpose of the Linux kernel's CPU-idle-time-management code, which is to do nothing efficiently. It is based on the observation that, if a logical CPU, which can be a symmetric multi-threading (SMT) thread or a core if SMT is not enabled, becomes idle (that is, there are no tasks to run on it), there is an opportunity to stop it. That will reduce the processor's power consumption, which generally allows some energy to be saved. Of course, that requires hardware support, but the majority of modern processors provide it. 

The CPU scheduler directs idle CPUs to execute a special piece of code, called the idle loop, whose role is to utilize processor capabilities to reduce power, if possible. For this purpose, the idle loop invokes the cpuidle subsystem in every iteration. That subsystem consists of three parts: the core that coordinates operations and provides an interface to user space, the driver that uses the processor interface (platform-specific in the majority of cases) to stop the given CPU and reduce the processor power, and the governor responsible for deciding how far the driver can go with the processor power reduction. 

Each iteration of the idle loop first checks if the CPU still has no tasks to run and if so, it invokes the cpuidle governor to make its decision, which includes deciding whether to stop the scheduler tick on the given CPU to prevent it from being woken up unnecessarily. Next, it calls into the driver to change the processor state in accordance with the decision made by the governor. The CPU running that code, which is idle from the scheduler's perspective, is stopped and the processor power can be reduced. Eventually, the CPU receives a wakeup event (for example, an interrupt) and it goes back to the scheduler or enters the next iteration of the idle loop. 

The decisions made by the cpuidle governor are based on the information about the processor's capabilities, which is supplied by the cpuidle driver in the form of a list of so-called "idle states". Each of those states represents a configuration with reduced power that the processor can be put into after stopping the given CPU. States are characterized by two parameters: the target residency (the minimum time to spend in the given idle state for which selecting it makes sense) and the exit latency (the worst-case time the processor will take to allow the CPU to execute instructions again). The list of idle states supplied by the driver is sorted by both the target residency and exit latency in increasing order. The idle states with lower values of those parameters are referred to as "shallower", whereas the idle states following them in the list order are referred to as "deeper". 

The problem is that the governor does not actually know how long the CPU will be idle, but it needs to select an idle state to match the upcoming period of CPU idleness, which is referred to as the idle duration. Doing that accurately every time would require a crystal ball, the supply of which is quite limited. So the governor has to resort to using statistics, and its decisions will never be perfect, though the existing governors strive to make accurate decisions in the majority of cases. The question is then which side it is better to err on, the "shallower" one, or the "deeper" one. 

Clearly, if the target residency of the selected idle state exceeds the idle duration (that is, an overly deep idle state is selected), it will hurt both energy efficiency and performance, because more energy could be saved by using a shallower idle state and the latency introduced into the workload would be lower. On the other hand, if the target residency of the selected idle state is too short, it will, again, hurt energy efficiency, because selecting a deeper idle state would allow more energy to be saved, but performance will not suffer. In theory, choosing a shallower state should even improve performance because the exit latency of the shallower idle state is lower than the exit latency of the deeper one. 

This seems to indicate that erring on the shallower side is slightly better and that, according to Wysocki, was the consensus in 2019 when that topic [was last discussed](<https://lwn.net/Articles/793372/>) at the OSPM Summit. However, more recently, it turned out that there were systems where selecting overly shallow idle states also hurt performance. 

The exact mechanism leading to that result on the systems in question (Chromebooks based on exotic x86 SoCs) has not been identified precisely. It has been observed, though, that using deep idle states on them allows the frequency of non-idle CPUs to go higher than when shallow idle states are used, and the busy CPUs are power-constrained. This most likely means that the power reduction resulting from idle states does not actually lower the overall energy usage of the system; instead, it allows the energy saved by idle CPUs to be utilized to crank up the frequency of the CPUs that are still executing instructions. In other words, what was designed as an energy-efficiency optimization has become a power-distribution mechanism, which had never been anticipated. 

With hindsight, this should have been expected, though, because power-constrained systems are abundant. In fact, the vast majority of client platforms shipping today are power-constrained, mostly due to thermally challenging form factors, so this problem may actually be more widespread than it would seem. 

This appears to pose a challenge to CPU-idle-time management in Linux; Wysocki admitted that he did not really know how to address it at the moment. Nevertheless, he thought that bringing it to everyone's attention would be useful. 

**Video:** [Success criteria for CPU idle state selection - Rafael J. Wysocki (OSPM26)](<https://youtu.be/0oE9TXescUc>)

#### Asymmetric packing for energy-efficient scheduling on Intel processors — Ricardo Neri

At OSPM 2025, Wysocki [presented](<https://lwn.net/Articles/1020596/#eas>) an energy-aware-scheduling (EAS) implementation for Intel hybrid processors. His energy model is based on the observation that E-cores (slower CPUs focused on energy efficiency) are more efficient than P-cores (faster, performance-oriented CPUs) across a wide portion of the power-performance curve. A fundamental limitation is that EAS in its current form requires prospective operating-point computations to be carried out at wakeup time, which is incompatible with hardware-controlled performance scaling, the predominant frequency scaling mechanism on Intel platforms. 

The presented alternative approach avoids per-wakeup cost computations and instead operates in the load balancer. It uses asymmetric task packing, giving E-cores higher placement priority while relying on the existing capacity-aware scheduling to keep heavy tasks on P-cores. The results from [schbench](<https://lwn.net/Articles/725238/#schbench>) showed comparable energy consumption to EAS with significantly lower 99th percentile latency. 

Vincent Guittot noted that EAS latency stems from its packing behavior and can be mitigated by tuning the scheduler slice time. 

Peter Zijlstra opposed introducing a parallel energy-efficiency mechanism, arguing instead for extending EAS. He suggested removing the incompatibility with hardware-controlled performance scaling, addressing packing-induced latency, and scaling to systems with large CPU counts. 

**Video:** [Asymmetric packing for energy-efficient scheduling on Intel processors - Ricardo Neri (OSPM26)](<https://youtu.be/zNd63UJX0gM>)

#### Rotating task scheduler in the Linux fair scheduling class — Pierre Gondois

Some workloads split their work into as many threads as there are CPUs on a platform. The workload completes only once all threads have finished. However, if the work partitioning is static, meaning each thread has the same amount of work to execute, and the platform is asymmetric (also known as heterogeneous multi-processing or HMP), the thread running on a big CPU will finish earlier than the one running on a little CPU. Some performance improvement could be had with a smarter load distribution. 

This highlights a more generic issue in the Linux fair scheduler: forward progress is not monitored across CPUs. A similar issue exists on SMP systems; with three long-running tasks running on two CPUs, the two tasks sharing one CPU will progress more slowly than the third task, which runs alone on the other CPU. 

The load balancer balances fair tasks between CPUs. For long-running tasks, load is analogous to the task's nice value: the lower that value, the higher the load, and the more running time a task should receive. By balancing the load between CPUs, tasks should receive approximately the amount of forward progress they deserve. 

In the examples above (long-running tasks on HMP, and three long-running tasks on two SMP CPUs), the load balancer doesn't detect and cannot solve the imbalance. Indeed, balancing is done from a CPU point of view rather than from a task point of view. 

The most accurate way to distribute forward progress fairly between tasks would be to use a global virtual run time. However, this solution does not scale. It would require a global [red-black tree](<https://lwn.net/Articles/184495/>) that would be accessed concurrently by all CPUs on the platform. 

The approach presented was to reinstate the initial contribution-scaled [per-entity load tracking](<https://lwn.net/Articles/531853/>) (PELT) implementation. The current scale-invariant PELT implementation makes it possible to estimate the size of a task uniformly, independently of the capacity of the CPU doing the estimation. The old contribution-scaled PELT, instead, measures the amount of instruction throughput received by each task. 

By adding another balancing mechanism that relies on the old throughput-based PELT implementation to estimate a forward-progress metric and swaps long-running tasks that don't progress at the same rate, the HMP-specific case was improved. The amount of time saved is platform specific. Unfortunately, the extra logic required, the HMP-specific aspect of the solution presented, and the fact that most multi-threaded workloads use dynamic partitioning make this solution unsuitable for upstream. Scientific workloads are better candidates for static partitioning, but are unlikely to run on HMP. 

**Video:** [Rotating task scheduler in Linux Fair scheduler class - Pierre Gondois (OSPM26)](<https://youtu.be/fdRLLqlrQp0>)

#### What's missing in sched_ext? — Andrea Righi

Sched_ext (SCX) is a Linux kernel framework that allows the CPU scheduler to be replaced at run time using BPF programs. While SCX has matured quickly and is now widely deployable, several architectural and integration challenges remain under active discussion. 

The long-troubled `ops.dequeue()` callback, broken until the 7.1 kernel release, has finally been fixed, giving BPF schedulers reliable visibility into when tasks are removed from their control by the core scheduler. That change is especially important for schedulers maintaining custom task queues or implementing their own accounting mechanisms in BPF. 

Another major improvement is the introduction of a dedicated deadline server for SCX tasks. Previously, aggressive `SCHED_FIFO`/`SCHED_RR` or heavily loaded fair-class workloads could starve SCX entirely, often triggering watchdog failures blamed on the BPF scheduler itself. The new mechanism reserves CPU bandwidth for SCX tasks and also enables "partial mode", where SCX and fair-scheduler tasks can safely coexist on the same CPUs without static partitioning. 

Some cleanup work is still needed around the deadline-server infrastructure itself. In particular, the kernel currently relies on statically configured bandwidth reservations made at boot time. The next step is to automatically register and unregister those reservations when an SCX scheduler is enabled or disabled. The discussion also touched on moving the debugfs interface used to configure values for the run time and period for the different deadline servers into sysfs (since debugfs may be unavailable on systems with secure boot or kernel lockdown). Zijlstra argued that it was too early to commit to a stable ABI before the FIFO control-group rework is better defined. 

The discussion also covered future work around hierarchical scheduling. [Initial support for per-control-group sub-schedulers](<https://lwn.net/Articles/1056014/>) which landed in the 7.1 release, allowing different scheduling policies to coexist across containers or workloads. The question was raised whether this hierarchy belongs in the kernel at all, with concerns that coordinating multiple schedulers with different latency and bandwidth constraints may ultimately require a single global scheduling model. 

Integration with [proxy execution](<https://lwn.net/Articles/953438/>) remains unfinished as well. The next step is to resolve the remaining configuration conflicts, allowing distribution kernels to be built with both proxy execution and sched_ext support enabled simultaneously, removing the current mutual-exclusion constraint. 

Another unresolved problem is evaluation: SCX now ships with a growing number of schedulers, but developers still lack good tools or benchmarks to explain why one scheduler performs better than another for a given workload. As the subsystem moves beyond experimentation into broader deployment, that question may become as important as the remaining kernel work itself. 

**Video:** [What's missing in sched_ext? - Andrea Righi (OSPM26)](<https://youtu.be/gftTDoV2Nuc>)

#### Sched_ext overheads and caveats — Christian Loehle

Loehle introduced sched_ext from a scheduler developer's perspective, including `sched_ext_ops` callbacks, local and global dispatch queues (DSQs), and custom DSQs. Terminal DSQs such as the local and global DSQs, hand control back to the core sched_ext machinery, while custom DSQs let the BPF scheduler retain ordering and placement control. Much of the real policy often ends up in `ops.dispatch()`, while wakeup-side callbacks, such as `select_cpu()` and `enqueue()`, need to stay cheap. 

Loehle then covered overhead measurements. A simple futex wait/wake test showed that callback structure matters a lot. Inserting a task directly into the local DSQ from `select_cpu()` could beat the fair scheduler on that microbenchmark, while always going through both `select_cpu()` and `enqueue()` was significantly slower. The default empty sched_ext path was much faster still, showing the cost of BPF callbacks rather than policy. Further measurements looked at the cost of tracking task state for PELT-like accounting: tracking of runnable and quiescent states was visible on hackbench and CPU-bound tests, run-time callbacks added little more, and `tick()` callbacks were more expensive, especially at 1000Hz. 

The later part of the talk was about where the current interface is still uncomfortable to use. Implementing PELT, capacity-aware scheduling, and EAS requires accurate knowledge of runnable, running, sleeping, migrating, and stolen-time states, which becomes hard once tasks disappear into terminal DSQs, especially the global DSQ. Misfit migration can be approximated with mechanisms such as `scx_bpf_reenqueue_local()`, or by expiring a task's slice from `tick()`, but these approaches add races and bookkeeping problems. Core scheduling and gang scheduling can also be expressed with custom DSQs, cookies, inter-processor interrupts, and locking, but coordinating sibling CPUs or switching a gang atomically remains awkward; new 7.1 features such as paired enqueue/dequeue callbacks and `SCX_ENQ_IMMED` help but do not solve everything. 

The discussion was brief and mostly about the practical consequences of those caveats. One topic was cache-line bouncing on shared DSQs, where DSQ granularity becomes part of scheduler design, for example by using a queue per-cluster domain. Another topic was whether the fastest benchmark represented a real policy; it was clarified as essentially the default empty sched_ext scheduler using the global DSQ. The EAS discussion covered stale per-CPU accounting, races when re-enqueueing local queues, and whether iterating the local DSQ would be any better. The final discussion topic was scheduler testing; sched_ext cannot directly test fair-scheduler-only code, but it can create deterministic scenarios for shared infrastructure such as PELT, core scheduling, or future mechanisms like proxy execution. 

**Video:** [Sched_ext overheads and caveats - Christian Loehle](<https://youtu.be/Wdx8TZwak9o?si=kgeBkM1demXiMokS>)

#### FlexGuard vs. time-slice extension: handling lock holder preemption — Victor Laforet

[FlexGuard (SOSP'25)](<https://dl.acm.org/doi/10.1145/3731569.3764852>) is a non-heuristic synchronization technique that uses eBPF to monitor context switches and detect critical-section preemptions. When a lock holder is preempted, FlexGuard proactively transitions waiting threads from spinning to blocking, freeing CPU resources to quickly resume the preempted critical section. By reacting to actual execution events rather than static thresholds, FlexGuard can improve performance by up to six times compared to POSIX mutexes. 

The [time-slice extension](<https://lwn.net/Articles/1038235/>) approach, often discussed in the Linux community, instead aims to prevent lock-holder preemption altogether by giving tasks a little CPU time to complete a critical section. Starting from Linux 7.0, a thread can use [`rseq()`](<https://manpages.opensuse.org/Tumbleweed/librseq-devel/rseq.2.en.html>) to efficiently notify the kernel when it holds a lock. Rather than preempting such a thread, the scheduler allows it to run until the lock is released (for at most 5µs by default, or up to 50µs), preserving forward progress. After a comparative evaluation of both solutions, on database indexes, LevelDB, and other benchmarks, we find that the two approaches address different bottlenecks and complement rather than replace each other. 

There was a discussion about realtime tasks and the chance of livelock. However, spinlocks should not be used with realtime tasks. And time-slice extension is not available with `PREEMPT_RT` anyway. Another discussion was about the fact that a spinlock should never be used by a large number of threads. Spinlocks should instead be fine-grained. This would allow for the use of basic spinlocks instead of queue locks, which are FIFO and have problems when the next waiter is preempted. The time-slice extension feature has been designed around fine-grained locks. However, some widely used software still uses broad locks and benefits from the increased throughput of queue-based spinlocks. 

**Video:** [FlexGuard vs. Time-Slice Extension: Handling Lock Holder Preemption - Victor Laforet](<https://youtu.be/aBNSTS9XH_4?si=QjlDCS0CBSO1zrI2>)
