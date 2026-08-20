---
title: "Reports from OSPM 2025, day two"
url: https://lwn.net/Articles/1021332/
date: "May 23, 2025"
category: "Scheduler-and power management"
author: "By Jonathan Corbet May 23, 2025 OSPM"
---

By **Jonathan Corbet**  
May 23, 2025

* * *

[OSPM](<https://lwn.net/Archives/ConferenceIndex/#OS-Directed_Power-Management_Summit-2025>)

The seventh edition of the [Power Management and Scheduling in the Linux Kernel Summit](<https://retis.sssup.it/ospm-summit/>) (known as "OSPM") took place on March 18-20, 2025\. Topics discussed on the second day include improvements to device suspend and resume, the status and future of sched_ext, the scx_lavd scheduler, improving the efficiency of load balancing, and hierarchical constant bandwidth server scheduling. 

As with the coverage from [the first day](<https://lwn.net/Articles/1020596/>), each report has been written by the named speaker. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Device suspend/resume improvements

Speaker: Rafael J. Wysocki ([video](<https://youtu.be/j-lgYOoAb00?si=B-bPq2Is3eGu5iIG>)) 

Possible improvements to device suspend and resume during system-wide power-management (PM) transitions were discussed. To start with, Wysocki said that this topic was not particularly aligned with the general profile of the conference, which focused on scheduling and related problem spaces, but he thought that spending some time on it might be useful anyway. It would be relatively high-level, though, so that non-experts could follow it. 

He provided an introductory part describing the design of the Linux kernel's code that handles transitions to system sleep states and back to the working state, and the concepts behind it. 

A system is in the working state, he said, when user-space processes can run. There are also system states, referred to as system sleep states, in which user space is frozen and doesn't do any work; these include system suspend and hibernation. The system enters sleep states to save energy, but when user work needs to be done, it goes back to the working state. Those transitions, referred to as system suspend and resume, respectively, affect the system as a whole and, if the kernel is configured to support system sleep states, every system component needs to play its part in handling them. In other words, support for system suspend and resume (and hibernation, if the kernel is configured to support it) is mandatory. 

As a rule, transitions from the working state into one of the sleep states are initiated by user space, but transitions from a sleep state back into the working state are started in response to a signal from a device; this signal is referred to as a system wakeup event. Devices allowed to trigger system wakeup events are referred to as wakeup devices. 

When a transition into a system sleep state is started, all devices need to be suspended. All activity must be stopped, hardware needs to go into low-power states, and wakeup devices need to be configured to trigger wakeup events. During a transition back into the working state, the reverse needs to happen, except that, in some cases, it is possible (or even desirable) to leave a device in suspend after a system resume and let it be handled by [run-time power management](<https://docs.kernel.org/power/runtime_pm.html>). All of that should be as fast as reasonably possible because some systems, like phones, suspend and resume often. 

In the working state, individual components of the system are subject to power management (PM) through frameworks like run-time PM, [device-frequency scaling (devfreq)](<https://docs.kernel.org/driver-api/devfreq.html>), [CPU-frequency scaling (cpufreq)](<https://docs.kernel.org/cpu-freq/index.html>), [CPU idling (cpuidle)](<https://docs.kernel.org/driver-api/pm/cpuidle.html>), [energy-aware scheduling (EAS)](<https://docs.kernel.org/scheduler/sched-energy.html>), [power capping](<https://docs.kernel.org/power/powercap/powercap.html>), and thermal control. Obviously, this needs to be taken into account when the system goes into a sleep state. Some devices may need to be reconfigured, which may require accessing their registers, and they may need to be resumed to satisfy dependencies. On the way back to the working state, care must be taken to maintain consistency with working-state PM. 

Dependencies between devices must be taken into account during transitions between the working state and sleep states. Obviously, children depend on their parents, but there are also dependencies between suppliers and consumers, represented in the kernel by device links. Dependent devices cannot be suspended after the devices they depend on and they cannot be resumed before those devices. 

Three layers of code are involved in transitions between the working state and sleep states of the system. The PM core is responsible for the high-level flow control, the middle-layer code (bus types, classes, device types, PM domains) takes care of commonalities (to avoid duplication of code, among other things), and device drivers do device-specific handling. As a rule, the PM core invokes the middle-layer code that, in turn, invokes device drivers, but in the absence of the middle-layer code, the PM core can invoke device drivers directly. 

There are four phases to both the suspend and resume processes. In the "prepare" phase of suspend, new children are prevented from being added under a given device and some general preparations take place, but hardware settings should not be adjusted at that point. As a general rule, device activity is expected to be stopped in the "suspend" phase; the "late suspend" and "suspend noirq" phases are expected to put hardware into low-power states. 

Analogously, the "resume noirq" and "early resume" phases are generally expected to power-up hardware. If necessary, the "resume" phase is expected to restart device activity, and the "complete" phase reverses the actions carried out during the "prepare" phase. However, what exactly happens to a given device during all of those phases depends on the specific combination of the middle-layer code and the device driver handling it. 

The "noirq" phases are so-called because interrupt handlers supplied by device drivers are not invoked during these phases. Interrupts are handled during that time in a special way such that interrupts involved in triggering wakeup events will cause the system to go back to the working state (resume). Run-time PM of devices is disabled during the "late suspend" phase and it is re-enabled during the "early resume" phase, so those phases can be referred to as "norpm" (no-run-time-PM) phases. 

The handling of devices during transitions between the working state and sleep states of the system is coordinated with device run-time PM to some extent. The PM core freezes the run-time PM workqueue before the "prepare" phase and unfreezes it after the "complete" phase. It also increments the run-time PM usage counter of every device in the "prepare" phase and decrements that counter in the "complete" phase, so devices cannot run-time suspend during system-wide transitions, although they can run-time resume during the "prepare", "suspend", "resume", and "complete" phases. 

Moreover, the PM core takes care of disabling and re-enabling run-time PM for every device during the "late suspend" and "early resume" phases, respectively. In turn, the middle-layer code and device drivers are expected to resume devices that cannot stay in run-time suspend during system transitions; they must also prevent devices that are not allowed to wake up the system from doing so. 

All of this looks kind of impressive, Wysocki said, but there are issues with it. At this point, he showed a photo of the Leaning Tower of Pisa, to the visible amusement of the audience. Fortunately, he said, the Linux kernel's suspend and resume code is safely far from collapsing. 

One of the issues that is currently being tackled is related to asynchronous suspend and resume of devices during system transitions between the working state and sleep states. 

Generally speaking, there are devices that can be handled out of order with respect to any other devices so long as all of their known dependencies are met; they are referred to as "async" devices. The other devices, referred to as "sync" devices, must be handled in a specific order that is assumed to cover all of the dependencies, the known ones as well as the unknown ones, if any. Of course, the known dependencies between the async and sync devices, represented through parent-child relationships or by device links, must be taken into account as well. 

Each of the suspend and resume phases walks through all of the devices in the system, including both the async and sync devices, and the problem is how to arrange that walk. For instance, the handling of all async devices may be started at the beginning of each phase (this is the way device resume code works in the mainline kernel), but then the threads handling them may need to wait for the known dependencies to be met, and starting all of those threads at the same time may stress the system. The processing of async devices may also be started after handling all of the preceding sync devices (this is the way device suspend code works in the mainline kernel), but, in that case, starting the handling of some async devices earlier may speed up the transition. That will happen if there are async devices without any known dependencies, for example. 

There are other possibilities, and the working consensus appears to be that the handling of an async device should be started when some known dependencies are met for it (or it has no known dependencies at all). The question that remains is whether or not to wait until _all_ known dependencies are met for an async device before starting the handling of it. 

Regardless of the way the ordering issue is resolved, the handling of the slowest async device tends to take the majority of the time spent in each suspend and resume phase. Consequently, if there are three devices, each of which happens to be the slowest one in a different suspend phase, combining all of the phases into one would reduce the total suspend time. Along these lines of reasoning, reducing the number of suspend and resume phases overall, or moving "slow" device handling to the phases where there is other slow work already, may cause suspend and resume to become faster. 

Another area of possible improvement is the integration of system transitions between the working state and sleep states with the run-time PM of devices. This integration is needed because leaving run-time suspended devices in suspend during system transitions may both save energy and reduce the system suspend and resume duration. However, it is not always viable, and drivers need to be prepared for this optimization so, if they want devices to be left in suspend, they need to opt in for that. 

Currently, there are three ways to do so: 

  * Participate in the so-called "direct-complete" optimization, causing the handling during a system suspend and resume cycle to be skipped for a device if it is run-time-suspended to start with. Hence the name; all suspend and resume phases except for "prepare" and "complete" are skipped for those devices, so effectively they go directly from the "prepare" to the "complete" phase. 
  * Set the `DPM_FLAG_SMART_SUSPEND` driver flag. 
  * Use [`pm_runtime_force_suspend()`](<https://elixir.bootlin.com/linux/v6.14.6/source/drivers/base/power/runtime.c#L1884>) as a system suspend callback. 

Unfortunately, the first option is used rarely, and the other two are not compatible with each other (drivers generally cannot do both of them at the same time). Moreover, some middle-layer code only works with one of them. 

Even if the driver opts in to leave the device in suspend, the device may still have to be resumed because of the wakeup configuration. Namely, run-time PM enables wakeup signaling for all devices that support it, so that run-time suspended devices can signal a need to take care of some event coming from the outside of the system. The power-management subsystem wants to be transparent and it doesn't want to miss any signal that may require the user's attention. 

On the other hand, only some of the wakeup-capable devices are allowed to wake up the whole system from sleep states, because there are cases in which the system needs to stay in a sleep state until the user specifically wants it to resume (for example, a laptop with a closed lid in a bag). For this reason, if a wakeup-capable device is run-time suspended prior to a system transition into a sleep state, and it is not allowed to wake up the system from sleep, it may need to be resumed and reconfigured during that transition. For some devices, the wakeup setting may be adjusted without resuming them, but that is not a general rule. 

Apart from the above, there are dependencies on the platform firmware and on other devices that may require a given device to be resumed during a system transition into a sleep state. Usually, middle-layer code knows about those dependencies and it will act accordingly, but this means that drivers generally cannot decide by themselves what to do with the devices during those transitions and some cooperation between different parts of the code is required. 

Leaving devices in suspend during a transition from a sleep state to the working state of the system may also be beneficial, but it is subject to analogous limitations. 

Drivers that don't opt in for the direct-complete optimization may need to specifically opt in for leaving devices in suspend during system resume. If they use use `pm_runtime_force_suspend()` as a suspend callback, they also need to use use [`pm_runtime_force_resume()`](<https://elixir.bootlin.com/linux/v6.14.6/source/drivers/base/power/runtime.c#L1945>) as a resume callback; this means that the device will be left in suspend unless it was in use prior to the preceding system suspend (that is, its run-time PM usage counter is nonzero or some of its children have been active at that time). If drivers set `DPM_FLAG_SMART_SUSPEND`, they also need to set `DPM_FLAG_MAY_SKIP_RESUME` to allow devices to be left in suspend. 

However, if a given device is not allowed to wake up the system from sleep, and it cannot be reconfigured without resuming, leaving it in suspend is not an option. Also, if the platform firmware powers up devices during system resume before passing control to the kernel, it is more useful to resume all of them and leave the subsequent PM handling to run-time PM. 

All of this needs to be carefully put in order. Different driver opt-in variants need to be made to work with each other and with all middle-layer code. Clear criteria for resuming run-time suspended devices during system transitions between the working state and sleep states need to be agreed on and documented, and all middle-layer code needs to adhere to them. In particular, `device_may_wakeup()` needs to be taken into account by all middle-layer code and in the absence of it, by device drivers and the PM core. 

In addition to the above, it can be observed that for all devices with run-time PM enabled, run-time PM callbacks should always be suitable for resuming them during transitions from system suspend into the working state unless they are left in suspend. In principle, some significant simplifications of device handling during system resume may result from this observation, but again this will require quite a bit of work. 

#### Sched_ext: current status, future plans, and what's missing

Speakers: Andrea Righi ([video](<https://youtu.be/YxO4g34qdB4?si=JPWIpMJCvXRJvlAv>)) and Joel Fernandes ([video](<https://youtu.be/ENCQZhpPiDU?si=5TaBBqsnbUhTI9V8>)) 

This talk covered the status of [sched_ext](<https://lwn.net/Articles/922405/>): a technology that allows schedulers to be implemented as BPF programs that are loaded at run time. The core functionality of sched_ext is now maintained in the kernel (after the merge that happened in 6.12) and it's following the regular development workflow like any other subsystem. 

Individual schedulers, libraries, and tooling are maintained in [a separate repository](<https://github.com/sched-ext/scx>). This structure was intentionally chosen to encourage fast experimentation within each scheduler. While changes still go through a review process, this separation allows a quicker development process. There is also a significant portion of this shared code base that is written in Rust, mostly topology abstractions and architectural properties that are accessible from user space and can be shared with the BPF code using BPF maps. 

The community of users and developers keeps growing and the major Linux distributions are almost caught up with the kernel and packages for the main sched_ext schedulers. 

An important question, raised by Juri Lelli, centered around the relationship with the kernel's completely fair scheduler (referred to here as "[`fair.c`](<https://elixir.bootlin.com/linux/v6.14.5/source/kernel/sched/fair.c>)") and whether it's worthwhile to reuse some of its functionality to avoid code duplication. In fact, sched_ext, being implemented as a new scheduling class, includes its own implementation of a default scheduling policy. BPF-based schedulers can then override this default behavior by implementing specific callbacks. The default implementation in sched_ext could just reuse parts of `fair.c` where appropriate to minimize code duplication and allow users to build on a base that closely mirrors the kernel's default behavior. 

However, reusing `fair.c` code is challenging due to its deep integration with various parts of the kernel scheduler. Features like energy and capacity awareness (EAS and CAS), which are not completely supported in sched_ext, complicate code reuse; introducing dependencies from sched_ext back into `fair.c` should be also avoided. 

Given these challenges, the consensus for now is to keep sched_ext independent by reimplementing similar functionality within its core. In doing so, the goal is to remain as consistent as possible with `fair.c`, with the possibility of converging toward a shared code base in the future. This approach also presents an opportunity to revisit and possibly eliminate some legacy heuristics embedded in `fair.c`, making it a potentially beneficial process for everyone. 

Another topic that was discussed is how to prevent starvation of `SCHED_EXT` tasks when a task running at a higher scheduling class is monopolizing a CPU. The proposed solution is to implement a [deadline server](<https://lwn.net/Articles/934415/>), similar to the approach used to prevent starvation of `SCHED_NORMAL` tasks. This work is currently [being handled by Joel Fernandes](<https://lwn.net/ml/all/20250315022158.2354454-1-joelagnelf@nvidia.com/>). 

One of the sched_ext key features highlighted in the talk is its exit dump-trace functionality: when a scheduler encounters a critical error, the sched_ext core automatically unloads it, reverting to the default scheduler, and triggering the user-space scheduler program to emit a detailed trace containing diagnostic information. This mechanism also activates if a task is enqueued to a dispatch queue (a sched_ext run queue), but is not scheduled within a certain timeout, making it especially useful for detecting starvation scenarios. 

Currently, there's no equivalent mechanism in `fair.c` to capture such traces. Thomas Gleixner suggested that we could achieve similar insights using tracepoints. Lelli added that, before the deadline server existed, the stalld daemon served a similar purpose: it monitored threads stuck in a run queue for too long without being scheduled, then temporarily boosted them using the `SCHED_DEADLINE` policy to grant them a small run-time slice. While the deadline server now can handle this in-kernel, stalld could still be used for its monitoring capabilities. 

A potential integration with cpuidle was also discussed, Vincent Guittot pointed out that we can just use the cpuidle quality-of-service latency interface from user space, which is probably a reasonable solution, as it just involves some communication between BPF and user-space and there's really no need to add a new specific sched_ext API for that. 

The talk also briefly touched the concept of tickless scheduling using sched_ext. A prototype scheduler (scx_tickless) exists; it routes all scheduling events to a designated subset of CPUs, while isolating the remaining CPUs. These isolated CPUs are managed to run a single task at a time with an effectively infinite time slice. If a context switch is needed, it is triggered via a BPF timer and handled by the manager CPUs using an inter-processor interrupt (allowing the scheduler to determine an arbitrary tick frequency, managed by the BPF timer). When combined with the `nohz_full` boot parameter, this approach enables the running of tasks on isolated CPUs with minimal noise from the kernel, which can be an appealing property for virtualization and high-performance workloads, where even small interruptions can impact performance. 

That said, the general consensus from the audience was that the periodic tick typically introduces an overhead that is barely noticeable, so further testing and benchmarking will be necessary to validate the benefits of this approach. 

Other upcoming features in sched_ext include the addition of richer topology abstractions within the core sched_ext subsystem and support for loading multiple sched_ext schedulers simultaneously in a hierarchical setup, integrated with cgroups. 

#### What can EEVDF learn from a special-purpose scheduler? The case of scx_lavd

Speaker: Changwoo Min ([video](<https://youtu.be/pHX-KgBCjwQ?si=cfavn-8NsRtxyZaq>)) 

Min gave a talk on a gaming-focused, sched_ext-based scheduler, scx_lavd (which was also [covered here](<https://lwn.net/Articles/991205/#lavd>) in September 2024). The talk started with a quick overview of the scx_lavd scheduler and its goals. Scx_lavd is a virtual-deadline-based scheduler (like [EEVDF](<https://lwn.net/Articles/925371/>)) specialized for gaming workloads. This approach was chosen because a virtual deadline is a nice framework to express fairness and latency in a unified manner. Moreover, by sharing a common foundation, there could be opportunities for the two schedulers to share lessons learned and exchange ideas. 

The technical goals of scx_lavd are achieving low tail latency (and thus high frame rates in gaming), lower power consumption, and smarter use of heterogeneous processors (like ARM big.LITTLE). He added that if scx_lavd achieves all three, it will be a better desktop scheduler, which is his stretch goal. 

He clarified that the main target applications are unmodified Windows games running on the Proton/Wine layer, so it is hard to expect additional latency hints from the application. An audience member asked if Windows provides an interface specifying latency requirements. Min answered that it does, and if a game or a game engine provides the latency hints, such information can be handed down to the scx_lavd through the Proton/Wine layer. 

Games are communication-intensive; 10-20 tasks are easily involved in finishing a single job (such as updating the display after a button press), and they communicate through primitives such as futexes, epoll, and [NTSync](<https://lwn.net/Articles/961884/>). A scheduling delay among one of the tasks can cause cascading delay and latency (frame time) spikes. 

The key question is how to determine which tasks are latency-critical. Min explained that a task in the middle of a task chain is latency-critical, so scx_lavd gives a shorter deadline to such a task, causing it to execute sooner. To decide whether a task is in the middle of a task chain, scx_lavd measures how frequently a task is blocked waiting for an event (blocking frequency) and how often a task wakes up another task (wakeup frequency). High blocking frequency means that the task usually serves as a consumer in a task chain, and high wakeup frequency indicates that the task frequently serves as a producer. Tasks with both high blocking and wakeup frequencies are in the middle of the chain somewhere. 

Participants asked about memory consumption (potentially proportional to the square of the number of tasks), the time to reach the steady state, how to decay those frequencies, and the relationship to proxy execution. Min answered that it simply measures the frequencies without distinguishing individual wakers and wakees, so it is pretty cheap. Those frequencies are decayed using the standard exponential weighted moving average (EWMA) technique, converging very quickly (a few hundreds of milliseconds) in practice. Also, compared to proxy execution, which strictly tracks a lock holder and waiters, scx_lavd's approach is much looser in tracking task dependencies. 

After explaining how scx_lavd identifies and boosts latency-critical tasks, Min showed a video [demo](<https://youtu.be/pHX-KgBCjwQ?t=1680>), of a game that achieves high, stable frame rates while running a background job. That led to further discussion about scx_lavd's findings. Peter Zijlstra mentioned that the determination of latency-critical tasks is something that could be considered for the mainline scheduler, but breaking fairness is not. 

Min moved on to how scx_lavd reduces power consumption. He is particularly interested in the system being under-utilized (say 20-30% CPU utilization) for running an old, casual game. He explained the idea of core compaction, which limits the number of actively used CPUs according to the system load, allowing inactive CPUs to stay longer in a deeper idle state and saving power. The relevance of EAS was discussed. Also, it was suggested that the core compaction needs to refer to the energy model for more accurate decisions on a broader variety of processors. 

#### Reduce, reuse, recycle: propagating load-balancer statistics up the hierarchy

Speaker: Prateek Nayak Kumbla ([video](<https://youtu.be/BOmGFv6Biuo?si=Ej6MeUyi8hUnQTEo>)) 

With growing core counts, the overhead of newidle balancing (load balancing performed when a CPU is about to enter the idle state) has become a scalability concern on large deployments. The past couple of years saw strategies such as [`ILB_UTIL`](<https://lwn.net/ml/all/cover.1690273854.git.yu.c.chen@intel.com/>) and [`SHARED_RUNQ`](<https://lwn.net/ml/all/20231212003141.216236-1-void@manifault.com/>) being proposed in the community to reduce the cost of idle balancing and to make it more efficient. This talk covered a new approach to optimize load balancing by reducing the cycles in its hottest function — [`update_sd_lb_stats()`](<https://elixir.bootlin.com/linux/v6.14.6/source/kernel/sched/fair.c#L11022>). 

The talk started by showing the benefits of newidle balancing by simply bypassing it; that made almost all the workloads tested unhappy. The frequency and the opportunistic nature of newidle balancing ensures that imbalances are checked frequently; as a result, the load is balanced opportunistically before the periodic balancer kicks in. 

`update_sd_lb_stats()`, which is called at the beginning of every load-balancing attempt, iterates over all the groups of scheduling domain, calling [`update_sg_lb_stats()`](<https://elixir.bootlin.com/linux/v6.14.5/source/kernel/sched/fair.c#L10343>) which, in turn, iterates over all the CPUs of the group and aggregates the load-balancing statistics. When iterating over multiple domains, which is regularly the case with newidle balancing, the statistics computed at a lower domain are never reused and are always computed over again, despite being done successively without any delay between them. 

The [new approach](<https://lwn.net/ml/all/20250313093746.6760-1-kprateek.nayak@amd.com/>) being proposed enables statistics reuse by propagating statistics aggregated at a lower domain when load balancing at a higher domain. This approach was originally designed to reduce the overheads of busy periodic balancing; Kumbla presented the pitfalls of using it for newidle balancing. 

Using the data from [`perf sched stats`](<https://lwn.net/ml/all/20250311120230.61774-1-swapnil.sapkal@amd.com/>) with the `sched-messaging` benchmark as the workload, it was noted that aggressively reusing statistics without any invalidation can lead to newidle balancing converging on the groups that are no longer busy. The data also showed a dramatic reduction in newidle balancing cost, which was promising. Even with a naïve invalidation strategy, the regression in several workloads remained, which prompted further investigation. It was noted that the `idle_cpu()` check in the scheduler first checked if the current running task is the swapper task. Newidle balancing is done prior to a context switch, and a long time spent there can confuse the wakeup path by making the CPU appear busy. Kumbla noted that perhaps the `ttwu_pending` bit can be reused to signal all types of wakeups and remove the check for the swapper task from the `idle_cpu()` function. 

Zijlstra noted that perhaps Guittot's [push task mechanism](<https://lwn.net/ml/all/20250302210539.1563190-7-vincent.guittot@linaro.org/>) can be used to redesign the idle and newidle balancing, and the statistics propagation can help reduce the overheads of busy-load balancing. Guittot mentioned an example implementation that uses a CPU mask to keep track of all the busy CPUs to pull from and idle CPUs to push tasks to. A [prototype push approach](<https://lwn.net/ml/all/20250409111539.23791-1-kprateek.nayak@amd.com/>) was posted soon after OSPM as an RFC to flesh out the implementation details. 

Zijlstra also noted that, during busy balancing, it is always the first CPU of the group that does the work for the domain, but perhaps that burden can be rotated among all the CPUs of the domain. There were some discussions on load-balancing intervals and how the statistics propagation would require aligning them for better efficiency. Kumbla noted that the prototype already contains a few tricks to align the intervals, but it could be further improved. 

Fernandes questioned whether the statistics can be still considered valid if tasks were moved at a lower domain. It was noted that reusing statistics should be safe for busy-load balancing, since only the load or the utilization is migrated, and the aggregates of these statistics will remain the same even if tasks are moved at lower domains. 

Julia Lawall asked if there have been any pathological cases where statistics propagation has backfired, to which Kumbla replied that the busy balancing is so infrequent compared to newidle balancing that it is very unlikely a single wrong decision will have any impact. Kumbla also requested for more testing to ensure that there are no loopholes in the logic. 

The talk went on to discuss a yet another strategy to optimize newidle balancing that introduced a fast path based on tracing the busiest CPU in the lowest-level cache (LLC) domain and, first, trying to pull the load from this CPU. It was noted that, despite yielding some benefit at lower utilization, the fast path completely fails when there are multiple concurrent newidle balance operations running and the lock contention at the busiest CPU leads to diminishing returns. 

The talk finished by discussing [SIS_NODE](<https://lwn.net/ml/all/168553468754.404.2298362895524875073.tip-bot2@tip-bot2/>) which expanded the search space of wakeup beyond the LLC domain to the entire NUMA node. It was noted that, despite looking promising at lower utilization, SIS_NODE quickly fails at higher utilization where the overhead of the larger search space is evident when it fails to find an idle CPU. A guard like SIS_UTIL is required as a prerequisite to make it viable but its implementation remains a challenge, especially in face of bursty workloads and an ever-growing size of the node domain. 

#### Hierarchical CBS with deadline servers

Speakers: Luca Abeni, Yuri Andriaccio ([video](<https://youtu.be/1-s8YU3Rzts?si=VvxLZUz75d75-xmC>)) 

This talk presented a new implementation of the hierarchical constant bandwidth server (HCBS), an extension of the [constant bandwidth server](<https://docs.kernel.org/scheduler/sched-deadline.html>) that allows scheduling multiple independent, realtime applications through control groups, providing temporal isolation guarantees. HCBS will allow realtime applications inside control groups to be scheduled using the `SCHED_FIFO` and `SCHED_RR` scheduling policies. 

In HCBS, control groups are scheduled through `SCHED_DEADLINE`, using [the deadline-server mechanism](<https://lwn.net/Articles/934415/>). Each group is associated with a bandwidth reservation (over a specified period), which is distributed among all CPUs. Whenever a control group is deemed runnable, the scheduler is recursively invoked to pick the realtime task to schedule. 

The proposed mechanism can be used for various purposes, such as having multiple independent realtime applications on the same machine, guaranteeing that they cannot interfere with each other, and providing access to realtime scheduling policies inside control groups, enforcing bandwidth reservation and control for those policies. 

The proposed scheduler aims at replacing and improving upon the already implemented [`RT_GROUP_SCHED` scheduler](<https://docs.kernel.org/scheduler/sched-rt-group.html>), reducing its invasiveness in the scheduler's code and addressing a number of problems: 

  * HCBS uses `SCHED_DEADLINE` and the deadline-server mechanism to enforce bandwidth allocations, thus removing all the custom code `RT_GROUP_SCHED` uses. The deferred behavior of the deadline server must not be used in HCBS, which is different from how deadline servers are used to enforce run time for `SCHED_OTHER` tasks. 
  * HCBS reuses the non-control-group code of the realtime scheduling classes to implement the local scheduler, with a few additional checks, to be as non-invasive as possible. 
  * The use of deadline servers solves the "deferrable server" issue of the `RT_GROUP_SCHED` scheduler. 
  * HCBS removes `RT_GROUP_SCHED`'s run-time migration mechanism. Instead, it only performs task migration. HCBS migrates tasks from CPUs that have exhausted their run time to others that still have available time. This allows it to fully exploit the allocated bandwidth. 
  * The HCBS scheduler has strong theoretical foundations. If users allocate an appropriate budget (computed by using realtime analysis), then it will be possible to guarantee respect for the application's temporal constraints. 
  * It also performs admission controls to guarantee that it can effectively provide the requested bandwidth. 

The current patchset is based on kernel version 6.13, but it is not complete yet. It passes most of the Linux Test Project tests and other custom-tailored stress tests. Tests with [rt-app](<https://github.com/scheduler-tools/rt-app>) are consistent with realtime theory. 

Arbitrary decisions on the implementation were discussed with the OSPM audience: 

  * The HCBS scheduler should only be available for the version-2 control group hierarchy. 
  * The bandwidth enforcement should not affect the root control group, to keep the current implementation of realtime policies. 
  * Tasks should only be allowed to run in leaf groups. Non-leaf control groups are only used to enforce partitioning of CPU time. 
  * Multi-CPU run-time allocation should follow the allowed CPU mask of the control group (`cpuset.cpus` file); disabled CPUs should not have run time allocated. 
  * The assignment of different run times for a given set of CPUs is currently done through the `rt_multi_runtime_us` knob, but reusing the standard `rt_runtime_us` knob has been suggested. 
  * Run-time migration of `RT_GROUP_SCHED` tasks has been removed to prevent over-commitment or CPU starvation. It has been suggested to look into solutions to perform such migration whenever possible to prevent unnecessary context switches.

As pointed out in the discussion, the scheduling mechanism may have counter-intuitive behaviors when over-committing: suppose a control group is allocated on two CPUs, each with 0.5 bandwidth usage, and two FIFO tasks are run, the first with priority 99 and usage of 0.8, the second with priority 50 and usage of 0.5, for a total usage of 1.3, over-committing the allocated bandwidth of 1.0. If the CPUs activate in parallel, both tasks will activate and will consume all the available bandwidth. The priority-50 task will use its requested bandwidth while the priority-99 task, even though it has higher priority, will consume only 0.5 out of the 0.8 usage. The result may also vary with a different distribution of the bandwidth on the same number of CPUs. 

An expected behavior, instead, would be that higher priority tasks must have higher priority on the total CPU bandwidth; in this case, the priority-99 task should always consume its bandwidth. Since these situations arise only when over-committing, thus outside theoretical analysis, they should not pose a problem.
