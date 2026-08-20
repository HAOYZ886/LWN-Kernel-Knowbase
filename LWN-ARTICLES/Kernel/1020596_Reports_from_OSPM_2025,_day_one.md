---
title: "Reports from OSPM 2025, day one"
url: https://lwn.net/Articles/1020596/
date: "May 19, 2025"
category: "Scheduler-and power management"
author: "By Jonathan Corbet May 19, 2025 OSPM"
---

By **Jonathan Corbet**  
May 19, 2025

* * *

[OSPM](<https://lwn.net/Archives/ConferenceIndex/#OS-Directed_Power-Management_Summit-2025>)

The seventh edition of the [Power Management and Scheduling in the Linux Kernel](<https://retis.sssup.it/ospm-summit/>) (known as "OSPM") Summit took place on March 18-20, 2025\. It was organized by Juri Lelli, Frauke Jäger, Tommaso Cucinotta, and Lorenzo Pieralisi, and was hosted by Linutronix at Alte Fabrik, Uhldingen-Mühlhofen, Germany. The event was sponsored by Linutronix, Arm, and the Scuola Superiore Sant'Anna in Pisa. 

The following contains summaries of the sessions; each summary is written by the session presenter. A recording of the entire summit is available [as a playlist](<https://www.youtube.com/playlist?list=PLohWCZQwiEVrpfcyHQ3s-lH_Yy0IiwlFT>) on the RetisLab YouTube channel. Photos of the event can be found on [this Google Photos page](<https://photos.app.goo.gl/oeEssZCrj8ZwfFS96>) or [as a 965MB zip archive](<https://drive.google.com/file/d/1eMmwyzJie-9sykdsGPZ42-oEaBnv4xLf/view?usp=sharing>). The [full set of slides from the sessions](<https://drive.google.com/drive/folders/1YsRMFjITS5SeenXlSkwynQyS6dc3TAC0?usp=sharing>) is also available. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Scheduling for core asymmetry and shared resources

Speaker: Morten Rasmussen ([video](<https://youtu.be/LFA-XeCqNeU?si=t8O5nqfpLebTjgU7>)) 

The Linux scheduler currently represents CPU throughput, by capacity and efficiency, through an energy model (EM), which was introduced with energy-aware scheduling (EAS). The EAS EM has been around for years and is quite simple relative to the complexity of modern systems. The EM, therefore, often fails to capture platform-specific constraints, leading to suboptimal performance and efficiency. The aim of the talk was to discuss potential ways forward to close the gap between the scheduler and power-management frameworks in the mainline kernel and the increasingly complex solutions developed by vendors out of tree. 

Previous studies have shown that power consumption and efficiency vary with the workload, which is not surprising, but the variation is significant and might be worth optimizing for. In mobile systems running Android, this and other aspects of optimization have been addressed by vendors with kernel modifications loaded in modules that attach functions to Android-specific vendor hooks. The number of vendor hooks in Android has grown to a level where most key scheduler mechanisms can be overridden by vendor modules. 

A look at a common [dynamic voltage and frequency scaling (DVFS)](<https://en.wikipedia.org/wiki/Dynamic_frequency_scaling>) implementation for mobile systems and expected upcoming implementations of DVFS with shared compute resources illustrates how the current CPU-frequency and scheduler abstractions fall short. In the first example, a subset of CPUs shares a clock frequency with the interconnect and L3 cache, which means that L3 cache DVFS scaling impacts CPU efficiency and performance directly. In the second example, DVFS scaling is further complicated by having a shared compute unit used transparently by all CPUs in the system. Modeling the clock-frequency dependencies of both examples and taking these into account for task placement is nontrivial. Often, simple platform-specific heuristics can help, while coming up with a generic, future-proof EM and EAS policy is virtually impossible. 

Two potential paths forward to improve mainline Linux were discussed: 

  * Improve EAS and EM to model the platform in more detail. This was considered mostly unrealistic. 
  * Work on expanding the architecture-specific callbacks from the scheduler to give architectures more influence on task-placement decisions. The scheduler already has a few callbacks to prioritize CPUs for certain systems. This could be extended to provide platform-specific preferred CPUs. 

In the discussion, the gap between the mainline, Android, and sched_ext communities was acknowledged. Adding architecture-specific callbacks seemed the most promising approach, although concrete proposals need to be shared for discussion. 

#### CAS/EAS on Intel client platforms

Speaker: Rafael J. Wysocki ([video](<https://youtu.be/n96cRZzPDtQ?si=E5V9UQAGypiWGB9G>)) 

Wysocki took the stage to discuss advances in implementing energy-aware scheduling on Intel hybrid chips. He started with a recap of two OSPM-summit 2024 presentations, one from himself and one from Ricardo Neri, that focused on the problems this effort was facing. Namely, the implementation of full-scale invariance on Intel hybrid processors was problematic, but has been overcome since last year's summit. There were also problems with creating energy models for those chips. The latter is still an issue but, fortunately, it can be addressed by using artificial energy models containing abstract cost values rather than any remotely realistic power numbers. He has been working on this problem recently. 

Next, Wysocki described scale invariance on the processors in question. There are two parts of it, capacity invariance and frequency invariance, each of which is implemented with the help of a function returning values between zero and `SCHED_CAPACITY_SCALE`, inclusive. One of them returns the CPU capacity relative to the capacity of the most performant CPU in the system and the other one returns the CPU frequency relative to the one at which the full capacity is reached. 

Generally, on x86 processors, the frequency-invariance part is derived from the values of the `APERF` and `MPERF` performance counters; this has been the case for several years. `MPERF` counts at a certain reference frequency, while `APERF` counts at the current frequency of the CPU. Snapshot them at two points in time, compute the deltas, divide the `APERF` delta by the `MPERF` delta, and you get the average CPU frequency relative to the `MPERF` counting frequency. However, this frequency needs to be relative to the frequency at which the full CPU capacity is reached. 

On hybrid Intel processors, the HWP (hardware P-state) interface is used and the frequency in question is proportional to the value of the `HIGHEST_PERF` field in the HWP capabilities (`HWP_CAP`) register, but that value is not in frequency units. It is an HWP performance level; the frequency corresponding to it can be obtained with the help of the performance-to-frequency scaling factor used by the intel_pstate driver for converting HWP performance levels to frequency values required by the CPU-frequency sysfs interface. This scaling factor is hard-coded in the driver for a few platforms but, in general, it can be derived from the information supplied by the [ACPI CPPC](<https://docs.kernel.org/admin-guide/acpi/cppc_sysfs.html>) interface. Even though this requires the platform firmware to get things right, intel_pstate is going to rely on it in the future. 

Since intel_pstate uses the performance-to-frequency scaling factor already, it can readily compute the frequency corresponding to `HIGHEST_PERF` and divide it by the `MPERF` counting frequency, which can be read from the `MSR_PLATFORM_INFO` register. The result of this computation is what is needed to turn the ratio of the `APERF` and `MPERF` deltas into the number required for frequency invariance. 

Once the frequency invariance has been taken care of, the capacity invariance is easy: multiply `HIGHEST_PERF` of the given CPU by `SCHED_CAPACITY_SCALE` and divide the resulting number by `HIGHEST_PERF` of the most performant CPU in the system. 

Scale invariance implemented in accordance with the above description has been used for enabling capacity-aware scheduling (CAS) on Intel hybrid processors and, by itself, it has led to some nice reductions in energy usage. It causes high-utilization tasks to be migrated to high-capacity CPUs when there is no room for them on the low-capacity CPUs, and it does not require assigning high priorities to high-capacity CPUs for this purpose. Thus low-utilization tasks can stay on low-capacity CPUs even when the higher-capacity CPUs are not fully utilized and energy can be saved. 

Of course, CAS is also required for supporting EAS, but the value added by the latter on top of the former turns out to be harder to demonstrate in practice than the value of CAS itself. Generally speaking, on low-power systems based on Intel hybrid processors, the difference made by EAS with respect to CAS appears to be in the noise. 

Nevertheless, according to Wysocki, work on enabling EAS on Intel hybrid processors has been in progress and it all boils down to finding a suitable artificial energy model that will cause the scheduler to prefer low-capacity CPUs to high-capacity ones until CAS starts to move tasks in the opposite direction. Basically, such an energy model needs to assign higher cost values to high-capacity CPUs and lower cost values to low-capacity ones regardless of their current frequency (or, generally, performance level). 

In the first energy model considered, one performance domain (PD) with only one performance level was created for each CPU type, so on a system with two types of CPUs there would be two PDs and all CPUs of the same type would belong to the same PD. The cost value associated with a given PD through its single performance level could be regarded as an inverse priority of the CPUs of the corresponding type. 

That energy model was as simple as it could be on the scheduler side, and it allowed some wakeup-path computations to be avoided, but more complexity was involved in creating it on the driver side. First of all, the driver needed to know the type of each CPU to add it to the right PD; it would also need to track the connections between CPUs and PDs in order to create CPU masks needed for PD registration. Next, the PDs might need to be expanded after they had been registered; that required some changes in the energy-model management code, which was not prepared for dealing with this use case. Finally, the energy-model management code expected all of the CPUs in one PD to be of the same capacity, which did not play very well with systems where the so-called favored cores were present (as those cores are just like the other cores of the same type, but their `HIGHEST_PERF` is higher, so additional HWP performance levels are available on them). 

Those difficulties did not prevent that energy model from being implemented in [the first EAS enablement for Intel chips patch series](<https://lwn.net/ml/linux-pm/5861970.DvuYhMxLoT@rjwysocki.net/>) posted near the end of 2024, and from being tested. The results, however, were not conclusive. Some improvements were observed, but mainly on high-power processors designed for desktop systems; on low-power systems, where energy usage improvements would be arguably more welcome, the difference made by EAS with respect to CAS alone was in the noise. Moreover, it was observed that EAS caused tasks to be migrated much more often between CPUs of the same type, which was attributable to using the highest spare capacity for CPU selection within one PD. 

That caused Wysocki to consider an alternative energy model in which PDs were created per CPU and the dominant cost component that still depended on the CPU type was amended by a small contribution proportional to the current performance level of the CPU. Four performance levels, corresponding to 100%, 80%, 60%, and 40% of the full capacity, were added to the energy model for each CPU, so that tasks would not be migrated to another CPU of the same type until the utilization of the CPU they were running on reached 40% of its capacity. In turn, if all of the CPUs of the same type were 40% utilized, tasks would start to be migrated between them if one of them became 60% utilized, and so on. The migration of tasks between CPUs of different types would still only happen by means of CAS. 

The alternative energy model was implemented in [the second EAS enabling for Intel chips patch series](<https://lwn.net/ml/linux-pm/22640172.EfDdHjke4D@rjwysocki.net/>) posted right before the summit, so it was new at the time of the conference and the evaluation of it was still in progress. 

As clearly demonstrated by Wysocki's work, EAS can be enabled on hybrid Intel hardware. However, it is somewhat heavyweight, so if the practical difference made by it continues to be hard to demonstrate convincingly, a different option may look more appealing. Specifically, what seems to be really needed is a way to move low-utilization tasks to low-capacity CPUs when there is spare capacity on them, to prevent the distribution of tasks in the system from remaining suboptimal for too long. It can be done by means of EAS, which is relatively expensive due to the EAS wakeup path overhead, but it can also be done with the help of a scheduler feature called "asym packing". 

Asym packing was originally introduced as a way of supporting favored cores. It is based on assigning priorities to CPUs and moving tasks from lower-priority CPUs to higher-priority CPUs during load balancing. It is used on multi-threaded Intel hybrid processors, where using CAS is not practical; in that case, CPU priorities are simply based on maximum capacity. The obvious drawback of doing so is that low-capacity CPUs are then only used when the high-capacity CPUs are fully utilized, so the energy usage tends to be higher than it needs to be. Still, in principle, asym packing can also be used for moving tasks in the reverse direction, but for this purpose it needs to be made capacity-aware and the CPU priorities need to be based on energy efficiency. 

Wysocki said that all of this could be done, so maybe asym packing would turn out to be a viable alternative to EAS, but the audience was skeptical about that prospect. The overall concern was that asym packing would not prevent tasks from being moved to CPUs that they should not be running on and taking them away from those CPUs later might not be sufficient to really improve the energy usage. 

#### Scheduler interface evolution

Speaker: Vincent Guittot ([video](<https://youtu.be/V5IHihhJbSI?si=M5nON3GvEiUHeFFI>)) 

The talk was about scheduler interface evolution, specifically focusing on the [`sched_setattr()`](<https://man7.org/linux/man-pages/man2/sched_setattr.2.html>) system call and how the parameters of run time, deadline, and period can be utilized to more effectively describe the behavior of different types of tasks. Guittot pointed out that these parameters are already used outside their original scope, the [deadline scheduler](<https://lwn.net/Articles/743740/>), as the run time is used to set a custom slice duration for [EEVDF](<https://lwn.net/Articles/925371/>). These three parameters can be used to describe several categories of tasks, and the parameters can be used by the scheduler and other frameworks to tune the behavior and the estimated utilization of the CPU. 

Guittot started describing a matrix for these task types: 

  * Fully periodic tasks are the most straightforward. They provide all information about their execution, but we don't often have such well-defined tasks. 
  * Sporadic, latency-sensitive tasks are characterized by having deadlines that are critical for responsiveness, but their arrival or the interval between their executions are not predictable. This is probably the most interesting type of task that enables setting a CPU bandwidth without providing a period. 
  * Sporadic running tasks only know their average running time but the period between two wakeups can fluctuate and its estimated utilization of the CPU as well. The running time can be translated into a utilization that will not vary with period. 
  * Random, latency-sensitive tasks might only have a critical deadline by which they must finish their execution. The scheduler might have little or no information about their expected run time or how frequently they will need to run, but it can use such information to set the maximum slice, as an example. 

A member of the audience pointed out that the scheduler often knows the deadline of a group of tasks but not always for individual tasks. The current proposal focuses on per-task attributes, since it doesn't seem easy to describe the attributes of a group of tasks if we can't even describe a single task. Graph algorithms should be evaluated when describing the groups of tasks. 

In addition to setting the slice, the scheduler could compute peak and/or average bandwidth for a task or better estimate its utilization of the CPU. It can even take into account a deadline and use these metrics when selecting the CPU. Those metrics are not that far from those used by the deadline scheduler; an audience member said that the first version of the deadline scheduler would fall back to the completely fair scheduler policy when tasks exhausted their bandwidth 

There were some concerns that user space can bias the values, but this is already the case with [uclamp](<https://lwn.net/Articles/762043/>). 

The presentation ended with some open issues related to the sum of estimated utilization which is often far higher than the maximum utilization of the CPU. 

#### Scheduler Governors

Speaker: Steven Rostedt ([video](<https://www.youtube.com/watch?v=VIyfhuEVmeQ>)) 

Rostedt led a session about scheduler governors, starting with a history of the Linux kernel scheduler that began with the original, single-run-queue scheduler. In 2002, Ingo Molnar created the [O(1) scheduler](<https://lore.kernel.org/lkml/Pine.LNX.4.33.0201040050440.1363-100000@localhost.localdomain/>), which had per-CPU run queues and some bitmask-based logic to find the next task to run in O(1) time. But it required heuristics to differentiate interactive and non-interactive tasks, so it could try to prioritize interactive ones. In 2004, Con Kolivas introduced "[plugsched](<https://lwn.net/Articles/109458/>)", a way to add different schedulers into the Linux kernel. Molnar was not pleased with this and [said](<https://lwn.net/Articles/109460/>): 

> I believe that by compartmenting in the wrong way we kill the natural integration effects. We'd end up with 5 (or 20) bad generic schedulers that happen to work in one precise workload only, but there would not be enough push to build one good generic scheduler, because the people who are now forced to care about the Linux scheduler would be content about their specialized schedulers. 

Later that year, Kolivas came back with [a new scheduler](<https://lwn.net/Articles/87729/>) called the "Staircase Scheduler" that replaced the heuristics with multiple run queues per CPU, where different queues had different priorities. As a task continued to run, it would move to lower-priority run queues and be preempted more often. Tasks that did not run on the CPU very long would stay in the higher-priority run queues and have lower latency. 

In 2007, Kolivas improved his scheduler into the [Rotating Staircase Deadline](<https://lwn.net/Articles/224865/>) (RSDL) scheduler, which added a quota for the entire level of a run queue. Tasks that slept through a quota-change period would not be affected by it, giving it better latency. Several people said at the time that Kolivas's scheduler gave a better user experience on a desktop environment and there was a push to get it upstream. 

During the debate about this scheduler, Molnar disappeared for three days and came back with a new scheduler called the [Completely Fair Scheduler](<https://lwn.net/Articles/230574/>) (CFS). This scheduler did away with run queues and used [red-black trees](<https://lwn.net/Articles/184495/>) instead. Tasks were added by their "wait time", and the scheduler code was broken up into separate files, making the interface a little more modularized. 

Kolivas left kernel development for a while, but then, in 2009, came back with the [Brain F*** Scheduler](<https://lwn.net/Articles/351499/>) (BFS). It used a single run queue for the entire machine (Kolivas said it would work fine up to 16 CPUs) and created a new, non-privileged scheduling class (`SCHED_ISO`) that ran between `SCHED_OTHER` and `SCHED_RT`, but was limited to how many tasks in that class could run. BFS implemented an earliest eligible virtual deadline first (EEVDF) algorithm, but did not support control groups. 

In 2011, Kolivas came back with a v2 of BFS that also introduced "skip lists". It had a O(1) algorithm for looking up the next task, a log(n) insertion, and made the scheduler a bit more topology aware. In 2016, Kolivas further improved the BFS scheduler, allowing it to scale better with more run queues; he called this scheduler the [Multiple Queue Skiplist Scheduler](<https://lwn.net/Articles/720227/>) (MuQSS). It implemented the BFS scheduler with a run queue on each CPU and was able to access the next task on other CPUs locklessly. It implemented an earliest effective virtual deadline first scheduler (loosely based on EEVDF). 

In 2022, Tejun Heo introduced [ext_sched_ext](<https://lwn.net/Articles/922405/>) (or sched_ext for short). This was a way to have pluggable schedulers using eBPF programs with the scheduling class of `SCHED_EXT`, which ran below `SCHED_OTHER`. To use it, one had to set all tasks to this scheduling policy. 

In 2023, Peter Zijlstra introduced a full-featured [Earliest Eligible Virtual Deadline First](<https://lwn.net/Articles/925371/>) (EEVDF). It replaced CFS, providing tasks with smaller deadlines with better latency. Eligibility (or the lack thereof) could still cause latency due to a task having run more than it should have. 

Rostedt then talked about scheduler constraints. A scheduler must be fair, and give equal CPU time to each task of the same priority. It is responsible for quickly scheduling tasks that have interactive requirements. The scheduler must respect a task's affinity for which CPU it may run on. It also affects the power efficiency of the system, especially if there are multiple types of cores to choose from. 

Rostedt then showed the graphs that were presented at the 2023 OSPM summit, which showed that, when comparing all the schedulers for a Chromebook work load, the MuQSS scheduler performed the best. Unfortunately, since the person who ran these benchmarks left Google and nobody has taken up the work recently, Rostedt didn't have any updated benchmarks for the latest EEVDF scheduler. 

Rostedt talked about scheduling being hard and complex. But can it be simplified if the scheduler could be focused on a specific type of system. Rostedt broke up the types of systems that a scheduler could be used for: 

  * Server: A system that deals with many users. 
  * Desktop: A system that deals with a single user interacting with many applications. 
  * Phone: A system that deals with a single user interacting with a single application. 
  * Embedded: A system with no users but just deals with event stimuli. 

Here is where Rostedt proposed a concept called "scheduler governors", making a point not to use the term "pluggable schedulers", as that usually implies changing the scheduler via a module or other out-of-tree source. He took the term from the power-management governors. The idea is that the governor would describe the system as a whole. A "server governor" would focus the scheduler to deal with multiple users, while a "desktop governor" would focus on scheduling tasks for a single user with many applications. If this structure had been in place when EEVDF came about, then one could have been able to easily benchmark EEVDF and CFS as they could have been implemented for different governors. Rostedt pointed out that sched_ext could still be used to tweak any of the governors. 

During the discussion, Rostedt described the difference between governors and pluggable schedulers like sched_ext. Sched_ext may enable the scenario that Molnar was afraid of, where everyone develops their own scheduler, and ideas would be more fragmented. A scheduler governor would be in the kernel proper, where developers focusing on the same environment could easily collaborate on the type of scheduler they needed. Server folks would focus on improving the server governor, where those who care about a phone could focus on the phone governor. It was brought up that there may only need to be two governors, as the desktop, phone, and embedded may not have such a different environment that different governors would be useful. But there was some agreement that a scheduler for a server has different requirements than a user device. 

It was mentioned that sched_ext could also be a governor, where the governor would give full control to a sched_ext eBPF program. 

Thomas Gleixner was still not convinced. He said that Apple has a quality-of-service interface that works fine and that Linux should do something similar. Rostedt agreed, but also mentioned that Apple doesn't deal with servers. There is an effort to create a library that would let tasks state their scheduling requirements and the library would translate that to the kernel. But making this work with all applications is difficult. 

Rostedt finished by saying that, at the bare minimum, he plans on working on cleaning up the scheduler code and make it more modular, where it could eventually have scheduler governors (at least out of tree perhaps). This work could also benefit sched_ext in the long run. 

#### Further uclamp sum aggregation

Speaker: Hongyan Xia ([video](<https://youtu.be/zTeD04gLDqw?si=TICrbHe4-go2IgC0>)) 

Xia proposed some improvements to the uclamp mechanism in a pre-recorded session viewed at the event. 

The current uclamp implementation, max aggregation, has several drawbacks that are being addressed in multiple proposed patch series. Hongyan's work, [uclamp sum aggregation](<https://lwn.net/ml/linux-kernel/cover.1741091349.git.hongyan.xia2@arm.com/>), attempts to address these problems by proposing a different implementation that may simplify the design while improving the effectiveness for uclamp as a performance hint. He presented the concept and design of sum aggregation in last year's OSPM. 

This year, with his experience on Android Dynamic Performance Framework (ADPF) and the interaction with applications, Hongyan has gathered further evidence and obtained new insights as to why sum aggregation makes sense in real-world scenarios like Chrome web browsing and virtual-machine CPU-frequency hints. To prove his claims, Hongyan presented benchmark results and energy numbers to show that the new implementation is more effective and energy-friendly. He also made a detailed decomposition to illustrate what part or parts of sum aggregation exactly contribute to the improved effectiveness. His findings add on top of last year's presentation to show that sum aggregation could be a nice alternative to the existing uclamp implementation as performance hints. 

#### Extending EAS with push callback

Speaker: Vincent Guittot ([video](<https://youtu.be/j-lgYOoAb00?si=B-bPq2Is3eGu5iIG>)) 

The talk addressed limitations in the existing EAS implementation and proposed additional changes on top of what is already under review on the mailing list. This presentation is a follow up on what has been presented at the Linux Plumbers Conference and the previous OSPM summit. 

The current EAS makes scheduling decisions only when a task wakes up, which can lead to suboptimal CPU selection as thermal conditions and system load change over time. The proposed solution introduces a push mechanism to re-evaluate task placement and possibly migrate tasks to more suitable CPUs. This mechanism is not used by the fair-scheduling class, with the exception of active migration of the running task, as mentioned by the audience. But the periodic load balance implemented by CFS can't scale for EAS because it implies checking all runnable tasks on all CPUs. Instead, the push mechanism looks for opportunities to migrate a task when it is put back in the enqueued list after being preempted. 

The current proposal is quite close to the mainline algorithm except that it adds a new level of decision for CPUs with the same cost level in a performance domain, which includes the maximum spare CPU capacity, the only parameter used currently. A member of the audience asked: why is the maximum spare capacity not enough? Utilization includes blocked load, which can make a CPU appear more loaded without any running tasks, as an example. 

Then, Guittot explained what he considered to be a "stuck task": one that doesn't have a wakeup event anymore, or which wakes infrequently because of uclamp, thermal constraints, or simple scheduling delay when sharing CPU time. The proposed condition is simple: it compares the `util_avg`, `util_ext` and `runnable_avg` of the task with the CPU capacity or its uclamp constraint. A participant asked why push is not used for other cases than EAS. Guittot has not considered other cases, like idle load balance (ILB), because there were no strong requests raised by the community so far. Zijlstra noted that ILB has always been an issue because it can be expensive on large systems; the push mechanism could be a solution for migrating tasks between low-level cache domains. 

In addition to [the current proposal](<https://lwn.net/ml/linux-kernel/20250302210539.1563190-1-vincent.guittot@linaro.org/>) on the mailing list, Guittot proposed to add more events to trigger the push of other types of tasks, such as a short-slice task that is being being preempted, a task with a changing profile, or just because it has run enough on the same CPU that the system conditions have changed. He mentioned 32ms, which is the half-life period, as a suitable value for "enough". 

Another proposal was to add more hints in the least-loaded-CPU selection, like the slice duration of a task in order to minimize the scheduling delay. Finally, the last change proposal is about biasing the energy decision; the energy delta between two performance domains is sometimes small, but can prevent migrating a task onto a CPU with a shorter scheduling delay. The problem remains to find a way to bias this decision without adding a new knob. 

At the end of his presentation, Guittot described a few open issues with the overutilized state, which is far too aggressive, and the run-to-parity feature, which can add significant scheduling delay when a short-slice task exhausts its slice. Then the discussion moved on discussing uclamp max versus sum and ended on this topic
