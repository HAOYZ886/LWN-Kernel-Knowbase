---
title: "Reports from OSPM 2025, day three"
url: https://lwn.net/Articles/1022054/
date: "May 30, 2025"
category: "Scheduler-Proxy execution; Scheduler-and power management"
author: "By Jonathan Corbet May 30, 2025 OSPM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 30, 2025

* * *

[OSPM](<https://lwn.net/Archives/ConferenceIndex/#OS-Directed_Power-Management_Summit-2025>)

The seventh edition of the [Power Management and Scheduling in the Linux Kernel Summit](<https://retis.sssup.it/ospm-summit/>) (known as "OSPM") took place on March 18-20, 2025\. Topics discussed on the third (and final) day include proxy execution, energy-aware scheduling, the deadline scheduler, and an evaluation of [the kernel's EEVDF scheduler](<https://lwn.net/Articles/925371/>). 

As with the coverage from [the first](<https://lwn.net/Articles/1020596/>) and [second](<https://lwn.net/Articles/1021332/>) days, each report has been written by the named speaker. 

#### Aren't you tired of proxy-execution talks?

Speaker: John Stultz ([video](<https://youtu.be/xcV1NtWENbs?si=I40Aq_jDkp3jBEUK>)) 

Continuing [what](<https://www.youtube.com/watch?v=FgB26cdJDZk>) has [been](<https://www.youtube.com/watch?v=wI36v6vCfkg>) a [persistent](<https://www.youtube.com/watch?v=HDtnNhLYsh0>) topic over the [last](<https://www.youtube.com/watch?v=QEWqRhVS3lI>) many [years](<https://www.youtube.com/watch?v=mlu9pC5IL2g>) at OSPM and the Linux Plumbers Conference (LPC), John Stultz gave a talk on [proxy execution](<https://lwn.net/Articles/934114/>), titled "Aren't you tired of proxy-execution talks?". While some in the audience kindly expressed feelings to the contrary, Stultz prefaced the talk by mentioning that, after a few years of doing these talks, _he_ was at least a little bit tired of the topic. 

The talk started with a quick summary of why proxy execution has been worth this extended effort: it resolves the priority inversion problems seen commonly with `SCHED_NORMAL` tasks when methods like [`cpu.shares`](<https://docs.kernel.org/scheduler/sched-design-CFS.html#group-scheduler-extensions-to-cfs>), CFS bandwidth throttling, [cpusets](<https://docs.kernel.org/admin-guide/cgroup-v2.html#cpuset>), or `SCHED_IDLE` are used to restrict background tasks so they don't affect foreground tasks. If one of those background tasks calls into the kernel and acquires a kernel mutex before it is preempted, it can block important foreground tasks from running if they also need that kernel mutex, causing unacceptable stalls that are visible to the user. 

Proxy execution solves this using a form of priority inheritance. Instead of tracking a linear priority order (which only works with realtime tasks), it keeps tasks waiting for a mutex on the run queue; if the important mutex-blocked task is selected to be scheduled, the scheduler finds the lock owner and runs that task instead. This safely enables the use of the above-mentioned features to restrict background tasks and, in a number of example tests, has shown an impressive reduction in outlier delays to the foreground tasks, providing more consistent behavior for users. 

Stultz highlighted some progress that has been made since [his 2024 LPC talk](<https://lpc.events/event/18/contributions/1887/>). The preparation patches that had been submitted repeatedly over months were finally merged. The [next chunk of the series](<https://lwn.net/ml/linux-kernel/20250516031814.1870508-1-jstultz@google.com/>), focusing on proxying only if the lock owner is on the same run queue as the waiter, had been submitted a few times and received helpful review feedback. Finally, the full proxy-execution patch series (as of v14) was merged into the android16-6.12 branch (disabled by default) so vendors could experiment and update their vendor hooks to support the concept of the split scheduling and execution contexts. 

As the full patch series is complex, Stultz has been trying to submit the series in smaller, reviewable chunks. He outlined the current stages of the full patch series, with the current one under review being the "single-run-queue proxying" step. That is followed by "proxy-migration and return-migration" then "sleeping owner enqueuing and activation" and finally "chain migration". 

Stultz covered the highlights from the [most recent (at the time of the talk) release (v15)](<https://lwn.net/ml/all/20250312221147.1865364-1-jstultz@google.com/>) of the patch set, some of the near-term to-do items, and the larger plans left for the year. Then, for discussion, he covered a topic he wasn't able to get to at the LPC talk last year, which is the complex logic required to handle sleeping-owner activation. When the proxy-execution logic walks the blocked-on chain and finds the lock owner sleeping, there's no way to effectively donate time and make the owner run. So the scheduler dequeues the donor and, instead, adds it to a list attached to the owner of the lock it's waiting on. When the lock owner wakes up, the scheduler can enqueue the blocked donor task(s) onto the same run queue, so proxying can commence. 

However, when [wait/wound mutexes](<https://lwn.net/Articles/548909/>) are involved, we can have "mid-chain" wakeups, which requires waking up and enqueuing just the tasks waiting on that mid-chain wounded mutex. Thus we have to maintain a full tree structure of blocked-on tasks. There is an additional complication, since the relationship is task-to-task, protected by the `task->blocked_lock`, and we need to take the `task->pi_lock` to wake a task. The locking is particularly complex, requiring that we drop and reacquire all the locks in each step. Making it all the more complicated, we can have standard unlock wakeups and multiple mid-chain wakeups happening in the tree in parallel. Stultz suggested that, when this chunk is submitted, he would welcome close review and suggestions for alternative approaches. 

Peter Zijlstra raised the point that the scheduler could just keep a single list and wake everything found there; that wouldn't be optimal, but hopefully this is a rare case in general. Steven Rostedt found this simpler approach attractive as well. Stultz was hesitant, as early versions of the patch used that approach, but it was a source of numerous bugs around wait/wound mutexes that have seemingly disappeared with the more complex approach. But he said he would evaluate it further. 

Thomas Gleixner pointed out that the `rt_mutex` logic has similar handling, so that should be looked at. Specifically, he was referring to the way the logic is documented, as it is also complex, and trying to add the documentation inline with the code makes it messy. Instead, Gleixner suggested explaining it all up front in a big comment, then using numbered annotations that can be added to the specific code. Others in the audience agreed that this was a nice way to document the complexity. 

Stultz was then able to move on to a grab bag of other topics that he has run into on the Android Systems team. 

The first was that the quality-of-service (QoS) efforts that were [much discussed at last year's OSPM](<https://www.youtube.com/watch?v=7j2E2tU49Gs&list=PLXJaBJgIrHE3g1hdEpypbFR4SHLEX6NfN&index=16>) were still important. He highlighted that [Qais Yousef's QoS API and ramp-up multiplier patches](<https://lwn.net/ml/all/20240820163512.1096301-1-qyousef@layalina.io/>) had shown to provide a nice (5-15%) performance improvement for Chrome on Android. In fact, when applied to an unoptimized 6.12 kernel, the results were similar to what is seen on a productized 6.1 kernel that includes all the device-specific vendor hooks and heuristics to improve scheduling behavior. More evaluation is needed to understand the relative power impact, but it seemed promising. 

Dietmar Eggemann raised the point that the series has quite a number of different changes together, and that it might be nice to split these apart to better understand which of the changes provided the benefit. He has mentioned this before, and Stultz clarified that he has made an attempt at pulling some of the changes apart, but found dependencies through the series that made this difficult. But he acknowledged the feedback and asked that folks try to help review the patches next time they are submitted. 

Another topic Stultz covered was Android's increasing use of [the freezer](<https://docs.kernel.org/power/freezing-of-tasks.html>) to restrict background tasks. Similar to proxy execution, this is a solution to prevent background tasks from impacting foreground tasks, but does so in a way that freezes background tasks in user space, where they can't hold kernel resources. The freezer is commonly used for suspend/resume and checkpoint/restore, but less frequently used on a per-task basis. 

This approach has shown promising results, but does have side effects. These include problems like synchronous binder calls to frozen tasks, which won't return. Binder buffers sent asynchronously have to be kept around and cannot be freed while the receiver is frozen. Efforts are ongoing to address those, but one other issue that has come up is problems with recurring timers and watchdog timers. Since a task can't run while it is in the freezer, when it is released from the freezer, any recurring application timers will fire repeatedly. This is similar to an issue seen in the early days with suspend and resume, which motivated the change causing the `CLOCK_MONOTONIC` clock ID to halt in suspend. Similarly, things like watchdog timers may fire while a task is frozen so, when it wakes up, it might panic because it wasn't runnable for such a long time. 

So there is a desire for a new, per-task clock ID that would not account time while the task was frozen. Stultz is hesitant about the idea, as it is not generalized yet, but wanted to raise the idea so folks were aware this type of use of the freezer was going on. There was some discussion in the crowd that, maybe, time namespaces could be used, though that seemed to get argued down. Gleixner highlighted that having a clock ID wouldn't be terrible, but the timer side would be difficult to manage. 

The next topic he covered was CPU-frequency-management trouble seen with realtime audio; he walked through a trace from the Android mainline kernel on a Pixel 6 device where a realtime task changed its CPU needs, but the CPU frequency didn't change quickly enough to avoid underruns (and more strangely the frequency only increased after the task had migrated and the CPU had gone idle). Vincent Guittot pointed out that, with mainline kernels, the CPU frequency should already be at a maximum and, despite android-mainline not having the scheduler vendor hooks enabled, there has to be some other logic in the Android patches causing this. 

Zijlstra asked why deadline scheduling isn't being used. Stultz clarified that efforts were made years ago to use it, but they were unsuccessful; he didn't have enough context on the details as to why. He agreed to dig further into the problem to understand the problems Android had with the deadline scheduler and to look deeper into the Android patches to see what has been done to frequency scaling on realtime tasks. 

Finally, he ended by suggesting that folks in the room try out [Perfetto](<https://perfetto.dev/>), which provides a great visualization tool for system behavior. Its traces (or images of the trace) can become an efficient sort of shorthand for describing problems or issues, which he thinks would be valuable for the community. Attendees asked how its functionality differs from [KernelShark](<https://kernelshark.org/>); Stultz explained that, while the visualization provided by Perfetto is useful, that part is similar to KernelShark, but the real power of Perfetto is that it ingests the trace into a database, which you can then query with SQL and render the results of that query into visualization. This makes it easy to identify where in a trace a problem occurred, and enables you to add your own visualization for metrics that aren't there by default. He mentioned that a lot of the documentation is Android or ChromeOS focused, but he has created a [document to show how to use it against mainline kernels](<https://gist.github.com/johnstultz-work/0ec4974e0929c4707bfd89c876ae4735>), and really hoped folks would take some time to try it out as it has a lot of potential to improve communication in the community. 

#### Improving energy-aware scheduling through better utilization signals

Speaker: Pierre Gondois ([video](<https://youtu.be/OwDrn1BxpWw?si=zofXR8ytP8wTZC1C>)). 

Utilization signals, which tell the scheduler how much CPU time any given task requires, are impacted by tasks sharing the same run queue. Indeed, in this case, the measured utilization doesn't necessarily represent the actual size of a task; it represents how much time the scheduler _gave_ to that task. In particular, if two tasks, A and B, are in the same queue, task A is perceived as idling when task B is running. 

On a fully utilized CPU, a task's `util_avg` value represents an underestimation of the actual size of the task. The `util_avg` of the task is bounded by the relative nice value of the task. A task with a low nice value will receive more running time and appear bigger than if it had a higher nice value. 

A prototype signal, `util_burst`, has been created to address this problem. This signal accumulates the amount of time a task ran while it was enqueued, then accounts the contributions all at once upon dequeuing. This allows co-scheduled tasks not to be perceived as idling when another task is scheduled. On a fully utilized CPU, the `util_burst` signal of a task is not bounded by the relative nice value of the task regarding the underlying CPU; it will slowly grow to reach the maximum value of 1024. 

This estimation of the size of tasks is related to the [`UCLAMP_MAX`](<https://docs.kernel.org/scheduler/sched-util-clamp.html>) feature. `UCLAMP_MAX` aims to put CPUs in a fully-utilized state. This rigs the estimation of the size of `UCLAMP_MAX` tasks and leads to inaccurate task placement. 

Indeed, `UCLAMP_MAX` tasks are now placed by the energy-aware scheduler (EAS), which uses utilization values to place tasks. On fully utilized CPUs, the load should be balanced among CPUs and utilization should be ignored. Thus the load balancer should be used in this case instead, or `UCLAMP_MAX` tasks should not be scheduled through EAS. 

#### 

#### Rethinking overutilized — when to bail out of EAS?

Speaker: Christian Loehle ([video](<https://youtu.be/N0tZ8GhhQzc?si=mBZx8M1njcnr6Vrz>)). 

The Linux scheduler's energy-aware scheduling mode makes per-task CPU placement decisions, based on [per-entity load tracking (PELT)](<https://lwn.net/Articles/531853/>) utilization signals, to place and pack tasks onto the most energy-efficient CPU. It's rather obvious that this isn't always desirable, so, when a run queue's utilization exceeds 80% of the CPU's capacity, EAS deactivates itself and falls back to traditional capacity-aware load balancing. This is motivated by the utilization data no longer being trustworthy when compute demand isn't met and potential energy savings being minimal in those scenarios. 

However, this mechanism is showing signs of age. The 80% threshold is static, independent of CPU type, and indifferent to workload behavior. On modern, heterogeneous systems, particularly mobile SoCs with complex core topologies or laptops with many big cores, this results in the system frequently bailing out from EAS. These transitions are often triggered by short-lived spikes that don't reflect sustained demand, but the global overutilized state impacts the entire system nonetheless. 

On the Pixel 6, which features clusters of little, mid-size, and big core, the little cores, with capacities of 160 (out of the normalized 1024), frequently cross the overutilization threshold under modest workloads. Since the 80% threshold translates to just 128 capacity on those CPUs, a transient bump is enough to deactivate EAS, even when big cores are idle and available. This leads to task migration from little to mid-size or big cores, then back again once demand drops, forming a cycle of unstable placements undermining energy savings. 

On desktop-class hardware, like the Apple M1 variants, which have more big cores than little cores, a problem manifests differently. A single compute-heavy thread on one big core can push it past the threshold, deactivating EAS across the entire system. For a mobile SoC with, traditionally, no more than two big CPUs (which can't both be sustained thermally), this was acceptable and rare; for a laptop use-case this can be improved upon. 

The fundamental issue around overutilization is that the scheduler cannot trust PELT's `util_avg` (and its derivatives) when compute demand exceeds the provided compute capacity. If a task is placed on an underpowered CPU and throttled, the task's load will be overestimated. 

Several proposals were discussed to address these issues. One is to make the overutilization threshold dynamic, as suggested by Yousef. Rather than a hard-coded 80%, the threshold would consider how much utilization a task could accumulate assuming it ran continuously until the next tick. This can account for differences between a 160-capacity little core and a 1024-capacity big core. While this introduces useful topology awareness, it risks backfiring on little cores by reducing their safe operating headroom, potentially triggering the overutilized state even sooner under normal bursts. 

A more conservative approach is to introduce a linger period before setting or clearing the overutilized flag. Since most overutilization events are short-lived, often under 1ms, delaying state transitions can filter out noise. Experimental results show that introducing a one-tick linger before clearing the flag reduces overutilized events by nearly an order of magnitude without significantly increasing time spent in the overutilized state overall. This could improve EAS stability with minimal side effects on responsiveness. 

Specifically, for topologies like those seen in laptops, if a big CPU is marked overutilized due to a single task, that core could be excluded from the global overutilized evaluation, since trustworthiness of PELT signals isn't impacted and maximum throughput for that task is already guaranteed. While this idea has limited value on mobile SoCs, it becomes relevant on desktops and laptops where lone-task domination is more common, and the performance budget is less constrained by thermal design. 

A more radical idea is to redefine the overutilized condition entirely, moving away from utilization metrics and, instead, basing the decision on observed idle time. If a CPU has not been idle within a recent window, say 32ms (the current PELT-half life), it may be assumed to be overutilized. This approach reflects actual compute demand more directly. 

Two issues arise. The approach is no longer [DVFS](<https://en.wikipedia.org/wiki/Dynamic_frequency_scaling>)-independent, meaning slow DVFS reactions or temporary DVFS restrictions may trigger the overutilized state. The bigger issue is that this is incompatible with `UCLAMP_MAX` since, whenever `UCLAMP_MAX` is actually taking effect, the condition is expected to be met (since provided compute capacity is artificially restricted). 

Here lies a deeper problem. The existence of `UCLAMP_MAX`, used on Android to restrict task-utilization estimations to save energy, undermines any mechanism that relies on utilization or idle inference. A task may run at 50% of CPU capacity, not because of limited demand, but because it has been constrained by user-space policy. In such cases, its utilization data is effectively garbage. Worse, any additional task scheduled on the same CPU contaminates its own utilization signal. The presence of `UCLAMP_MAX` therefore limits the design space of alternative overutilization definitions. 

One potential fix is to redefine `UCLAMP_MAX` as a hard cap on `util_avg`, effectively throttling the task by blocking or scheduling it out once it hits its maximum. This would make PELT trustworthy again, although at the cost of changing the semantics of `UCLAMP_MAX` in a way that may not be acceptable to existing users. It's unclear whether this is viable, especially given how heavily Android vendors depend on these interfaces through custom vendor hook implementations. 

The takeaway is clear: the current overutilized mechanism does not scale well with modern topologies or different use cases. Its reliance on a single global threshold and its entanglement with `UCLAMP_MAX` are all contributing to misbehavior across platforms. While approaches had been discussed previously, none of the proposals made it anywhere so far. 

Discussion revolved around the proposals discussed, limitations of PELT signals, but also the `UCLAMP_MAX` issue, more specifically having user-space policy that doesn't have obvious mainline users (but rather indirect downstream users with modified functionality) and how much these should be restricting future developments around the overutilized state. 

#### How to verify EAS and uclamp functionality?

Speaker: Dietmar Eggemann ([video](<https://youtu.be/9mMwinAopRs?si=aB-57xsIxtUfWFud>)). 

This talk addressed the lack of review for several EAS and uclamp patch sets on the mailing list. High commercial interest in Android, fragmented test environments, and vendor-specific customizations have hindered collaboration in this area for some time now. Traditional performance benchmarks are insufficient, as EAS and uclamp aim to balance both performance and energy efficiency. The workload generator [`rt-app`](<https://github.com/scheduler-tools/rt-app>), ftrace, and EAS/PELT kernel tracing were mentioned as tooling for this job instead. 

A proposed solution is to share `rt-app` configuration files that trigger the behavioral changes introduced by patches, along with instructions for adapting them to different hardware (e.g., CPU count, capacity, task parameters). This allows consistent testing across diverse heterogeneous platforms, while maintaining flexibility in data analysis. 

Arm has already adopted this method for [the uclamp sum aggregation patch series](<https://lwn.net/ml/all/cover.1741091349.git.hongyan.xia2@arm.com>), using JupyterLabs notebooks and the [LISA (Linux Integrated System Analysis) toolkit](<https://tooling.sites.arm.com/lisa/latest>). It should be clear to all of us that relying solely on testing during Android release cycles is insufficient and risks introducing regressions into mainline EAS/uclamp code. 

#### Making user space aware of current deadline-scheduler parameters

Speaker: Tommaso Cucinotta ([video](<https://youtu.be/T2siNuC9T60?si=jWgK1jBeLKMtyGiP>)). 

The `SCHED_DEADLINE` scheduler allows reading its statically configured run-time, deadline, and period parameters through the [`sched_getattr()`](<https://man7.org/linux/man-pages/man2/sched_setattr.2.html>) system call. However, there is no immediate way to access, from user space, the current parameters used within the scheduler: the _instantaneous_ run time, as well as the current absolute deadline. This information can tell a task how much more CPU time it has available to it in the current computation cycle. 

The talk described the need for this data in the context of adaptive realtime applications, with two use cases. The first deals with imprecise computations, which are quite common in control applications, where a realtime task, after performing its most important computation, may decide to execute an optional computation refining its output if enough time is available in the leftover run time until the deadline. The second use case deals with a [pool of `SCHED_DEADLINE` threads](<https://retis.santannapisa.it/~tommaso/papers/rtas25.php>) serving jobs from a shared queue, where each thread needs precise information on its own leftover run time and absolute deadline to estimate whether a new job can be picked and processed. 

The talk compared a few ways to read the current `SCHED_DEADLINE` parameters. 

  * With user-space heuristics, running the risk of "open-loop" measurements that might go out-of-sync with the real values used by the kernel. 
  * Using the values exposed through the `/proc/self/task/pid/tid/sched` special file, which contains lines for the `dl.runtime` and `dl.deadline` for `SCHED_DEADLINE` tasks. This approach is inefficient because of the need to format numbers in decimal notation in kernel space and parse them back in user-space. This approach also suffers from the problem of correctly interpreting, in user space, the absolute deadline information available in the `rqclock` reference. 
  * Using a kernel module that exposes the needed numbers more efficiently, in binary format, through a `/dev/dlparams` special device. However, this module does not have direct access to the deadline-scheduler API to update the leftover run time, so it may call `schedule()` for the purpose, which is probably overkill considering what is actually needed. Also, this would not work if the parameter were to be read from a different CPU than the one where the monitored `SCHED_DEADLINE` task is running. 

Finally, the talk presented a small kernel patch to `sched_getattr()`; it uses the `flags` parameter (which is mandated to contain zero at the moment) to enable a process to request its leftover run time and absolute deadline (converted to a `CLOCK_MONOTONIC` reference), instead of the statically configured parameters. 

Questions for the OSPM audience included: 

  * The suitability of the user-space interface, especially the addition of a flag to `sched_getattr()`; the feedback seemed positive in this regard. 
  * A possible option to return the current absolute deadline relative to `CLOCK_REALTIME` in addition to `CLOCK_MONOTONIC`: there seemed to be little value in adding such an option, as any realtime application should be designed not to rely on `CLOCK_REALTIME`. 
  * The possibility that the absolute deadline value returned could exhibit some nanosecond-level variability if multiple calls are made within the same deadline cycle. 
  * The possible drawback that reading the current parameters would cause a `SCHED_DEADLINE` task that exhausted its run time to be immediately throttled. Without such a call, with the default kernel configuration, throttling would have happened at the next clock tick. This didn't seem such a big problem, in view of the fact that a reasonable use of `SCHED_DEADLINE` would need to enable the `HRTICK_DL` scheduler feature, so in such conditions there would be no such difference in behavior. 
  * If the fact that `sched_getattr()` grabs the `rq->__lock` of the run queue, where the monitored `SCHED_DEADLINE` task resides, to call `update_rq_clock()` and `update_curr_dl()`, might constitute a problem when the task executing the system call is running on a different CPU. 

#### The EEVDF verifier: a tale of trying to catch up

Speaker: Dhaval Giani ([video](<https://youtu.be/iel9S0G9JJc?si=VCoCpTfn502NShp5>)) 

The talk started with the promise of (Swiss) chocolates for audience participation. Giani started by talking a bit about the [EEVDF technical report](<https://citeseerx.ist.psu.edu/document?repid=rep1&type=pdf&doi=805acf7726282721504c8f00575d91ebfd750564>), which directed the implementation of the kernel's EEVDF scheduler. He pointed out that this is the first time that the `SCHED_NORMAL` class of the CPU scheduler has been based on an academic paper and it brings about a few interesting ideas, the chief among them being around the idea of functional tests of the scheduler. For the longest time, the scheduler has been a bunch of heuristics around a core algorithm. 

He started by walking through the EEVDF algorithm with a simple case — showing how lag, deadline, and virtual time are calculated. With this background, the mathematical guarantees offered by the algorithm were discussed. These guarantees will form the basis of the functional tests for EEVDF. When prompted, Zijlstra agreed this was an adequate explanation of the paper for the basis of future discussions. 

With this, the focus shifted to start talking about how the Linux implementation differed from the paper. One of the major changes comes from the fact that EEVDF doesn't consider SMP (support for which was introduced in Linux in 1996, while the technical report came out in 1995). Another major difference is the existence of a hierarchy of control groups. Due to these reasons, it is not possible for there to be a global virtual clock, which is what the paper relies on. Instead, for each scheduler run queue, there is a "zero-lag" `vruntime` value that is used to keep track of where the virtual clock of the CFS run queue is. The Linux implementation also takes some liberties by using some of the mathematical properties to clamp down lag to reduce the numerical instability. 

Next, Giani talked about using the results expected from the lemmas and theorems introduced in the paper to test if the algorithm is still faithful to the paper, and if not, then document the differences introduced. Two test cases were covered, and there was consensus that they made sense, even if they would always pass. As previously mentioned, this is testing the functionality — and either of them failing would mean something fundamental has been changed. 

Zijlstra suggested that, for the purpose of testing, some of the clamps can be relaxed to see how unstable the math is around the `vruntime` updates. It will also help document in which cases the clamping is needed. 

It is still to be decided how these tests would be triggered, but they will need to implemented as a part of the kernel as they look into deep scheduler internals and there is no appetite to expose those internals outside the scheduler. 

As the talk would down, Giani pointed out that the two tests he had finished developing work quite well on the current implementation, congratulated Zijlstra, and offered him an additional chocolate for the hard work.
