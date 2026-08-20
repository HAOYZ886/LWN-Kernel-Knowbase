---
title: The Linaro Connect scheduler minisummit
url: https://lwn.net/Articles/482344/
date: "February 22, 2012"
category: Scheduler
author: "February 22, 2012 This article was contributed by Paul McKenney"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

February 22, 2012

This article was contributed by Paul McKenney

I had the privilege of acting as moderator/secretary for the recent Scheduler Mini-Summit at Linaro Connect, which was attended by Peter Zijlstra (Red Hat), Paul Turner (Google), Suresh Siddha (Intel), Venki Pallipadi (Google), Robin Randhawa (ARM), Rob Lee (Freescale assigned to Linaro), Vincent Guittot (ST-Ericsson assigned to Linaro), Kevin Hilman (TI), Mike Turquette (TI assigned to Linaro), Peter De Schrijver (Nvidia), Paul Brett (Intel), Steve Muckle (Qualcomm), Sven-Thorsten Dietrich (Huawei), and was ably organized by Amit Kucheria (Linaro). Rough notes from the session can be found [here](<https://wiki.linaro.org/WorkingGroups/PowerManagement/Doc/HMPscheduling>). 

The main goals of the mini-summit were as follows: 

  1. Take first step towards planning any Linux-kernel scheduler changes that might be needed for ARM's upcoming [big.LITTLE [PDF]](<http://www.arm.com/files/downloads/big.LITTLE_Final.pdf>) systems to work well (see also [Nicolas Pitre's LWN article](<http://lwn.net/Articles/481055/>)). 

  2. Create a power-aware infrastructure for scheduling and related Linux kernel subsystems. For example, integrate dyntick-idle, cpufreq, cpuidle, [sched_mc](<https://lwn.net/Articles/474915/>), timers, thermal framework, pm_qos, and the scheduler. 

  3. Provide a usable mechanism that reliably allows all work (present and future) to be moved off of a CPU so that said CPU can be powered off and back on under user-application control. CPU hotplug is used for this today, but has some serious side effects, so it would be good to either fix CPU hotplug or come up with a better mechanism--or, in the best Linux-kernel tradition, both. Such a mechanism might also be useful to the real-time people, who also need to clear _all_ non-real-time activity from a given CPU. 

How well did we meet these goals? Read on and decide yourself! To that end, the remainder of this article is organized as follows: 

  1. Overview of ARM big.LITTLE Systems
  2. Major Issues Considered
  3. Future Work and Prospects
  4. Conclusions 

Following this is the inevitable Answers to Quick Quizzes. 

###  Overview of ARM big.LITTLE Systems

ARM's big.LITTLE systems combine the [Cortex-A7](<http://www.arm.com/products/processors/cortex-a/cortex-a7.php>) and [Cortex-A15](<http://www.arm.com/products/processors/cortex-a/cortex-a15.php>) processors. Both processors are implementations of the ARMv7 architecture and they execute the same code. ARM stated the little Cortex-A7 design was focused on energy efficiency at the expense of performance. The bigger Cortex-A15 design was, instead, focused on performance at some cost to energy efficiency. In practice this means the little core will be somewhat quicker and a lot more power efficient than today's Cortex-A8: a multi-core configuration of these little cores could run today's smartphones. The big core will significantly outperform Cortex-A9 within a similar power budget. 

**Quick Quiz 1** : But what if there is a different number of Cortex-A7s than of Cortex-A15s? Answer

One way to use a big.LITTLE system is to have equal numbers of Cortex-A7 and Cortex-A15 CPUs paired up, so that only one CPU of a given pair is running at a time. This pairing is "a continuation of dynamic voltage/frequency scaling by other means". To see this, imagine the Cortex-A15 initially running at maximum clock frequency, with the voltage and frequency decreasing until the performance is barely greater than that of the Cortex-A7 CPU. At this point, the firmware switches the software context from the Cortex-A15 to the Cortex-A7, with the Cortex-A7 initially running at its maximum clock frequency, but at lower power than the Cortex-A15. 

**Quick Quiz 2** : Why scale down? Isn't it always better to run full out in order to race to idle? Answer

The voltage and frequency of the Cortex-A7 can then be further decreased, in turn further decreasing the power consumption. 

For some implementations, thermal limitations would require that the Cortex-A15 CPUs be used only for short bursts at maximum frequency, as was discussed at length at the summit. However, I have since learned that many other implementations are expected to be fully capable of running the Cortex-A15 CPUs at maximum frequency indefinitely. 

The switch between the Cortex-A7 and Cortex-A15 CPUs is implemented in firmware, but Grant Likely, Nicolas Pitre, and Dave Martin are moving this functionality into the Linux kernel. 

In many big.LITTLE designs, it is also possible to run both the Cortex-A7 and Cortex-A15 CPUs concurrently in an shared-memory configuration. However, this means that the Linux kernel sees the big.LITTLE architecture, which in turn raises the issues discussed in the next section. 

###  Major Issues Considered

Those of you who know the personalities in attendance will not be surprised to hear that the discussions were both spirited and wide-ranging. However, most of the discussion centered around the following four major issues: 

  1. Benchmarks and Synthetic Workloads
  2. Parallel Hardware/Software Development
  3. What Do You Do With a LITTLE CPU?
  4. CPU Hotplug: Kill It or Cure It? 

Each of these issues is covered in one of the following sections: 

####  Benchmarks and Synthetic Workloads

The biggest and most pressing issue facing SMP-style big.LITTLE systems is the lack of packaged Linux-kernel-developer-friendly benchmarks and synthetic workloads. C programs and `sh`, `perl`, and `python` scripts can be friendly to Linux-kernel developers, while benchmarks requiring (for example) an Android SDK or a specific device will likely be actively ignored. 

It is critically important for benchmarks to provide a useful "figure of merit", which should encompass both user experience and estimated power consumption. For example, a synthetic workload that models a user browsing the web on a smartphone might have a smaller-is-better estimate of average power consumption, but also have the constraint that the system respond to emulated browser actions within (say) 500ms. If the response time is within the 500ms constraint, then the figure of merit is the estimated average power consumption, but if that constraint is exceeded, the figure of merit is a very large number. The exact computation of the figure of merit will vary from benchmark to benchmark. 

Currently, some rough and ready workloads are in use. For example, Vincent Guittot used cyclic test in [his work](<https://wiki.linaro.org/WorkingGroups/PowerManagement/Doc/Hotplug>). While this did get the job done for Vincent, something more adapted to embedded/mobile workloads instead of real-time computing would be quite welcome. Zach Pfeffer of Linaro will be doing some workload creation in his group, however, given the wide variety of mobile and embedded workloads, additional contributions would also be welcome. 

Finally, the scheduler maintains a great number of statistics and tracepoints. A "schedtop"-style tool that provides a mobile/embedded view of this information would be very valuable. 

####  Parallel Hardware/Software Development

Even if you don't know exactly when a given piece of hardware will be available, it is a good bet that it will become available too late to get the needed software running on it. It is therefore critically important to have some way to develop the needed software _before_ the hardware is available. Thankfully, there are a number of ways to test big.LITTLE scheduler features before big.LITTLE hardware becomes available. 

One crude but portable method is to create a `SCHED_FIFO` thread on each LITTLE-designated CPU, and to have this thread spin, burning CPU, for (say) one millisecond out of every two milliseconds. This approach perturbs the scheduler's preemption points, particularly the wake-up preemptions. Nevertheless, this approach is likely to be quite useful. 

A less portable but more accurate approach is to constrain the clock frequency of the CPUs so that the big-designated CPUs have a lower bound on their frequency and the LITTLE-designated CPUs have an upper bound on their frequency. The way to do this is via the `sysfs` files in the `/sys/devices/system/cpu/cpu*/cpufreq` directories, the most pertinent of which are described below. 

**Quick Quiz 3** : I typed the following commands: 

```
cd /sys/devices/system/cpu/cpu1/cpufreq
      sudo echo 800000 > scaling_max_freq
```

Despite the `sudo`, I got "Permission denied". Why doesn't `sudo` give me sufficient permissions? Answer

Echoing a number into the `scaling_max_freq` file will require that the corresponding CPU's frequency be limited to the specified number in KHz. Echoing a number into the `scaling_min_freq` file will require that the corresponding CPU's frequency be at least the specified number in KHz. Reading the `scaling_available_frequencies` file will list out the frequencies (again in KHz) that the corresponding CPU is capable of running at. For example, the laptop I am typing on gives the following list: 

```
2534000 2533000 1600000 800000
```

Reading the `affected_cpus` file lists the CPUs whose core clock frequencies must move in lockstep with the corresponding CPU. On my laptop, each CPU's frequency may be varied independently, but it is not unusual for a given "clock domain" to contain multiple CPUs, which then must all run at the same frequency, for example, on systems with hardware threads. Reading the `scaling_cur_freq` file gives you the kernel's opinion on what frequency the corresponding CPU is running at. Reading the `cpuinfo_cur_freq` file, instead, gives you the hardware's opinion on what frequency that the corresponding CPU is running at, which might or might not match the kernel's opinion, so you should most definitely experiment to make sure that all of this is doing what you want on your particular hardware and kernel. 

For more information, see `Documentation/cpu-freq` in the Linux kernel source directory. 

There was also some discussion of ways that the `linsched` user-mode scheduler simulator might help with prototyping. 

Finally, it is possible to use T-states on Intel platforms to emulate a big.LITTLE system. According to Paul Brett: 

Intel Architecture processors provide a clock modulation control exposed as the MSR_IA32_THERM_CONTROL MSR. This MSR can be used to reduce the effective clock frequency for each core independently in 12.5% increments from 100% down to 12.5%. Under normal conditions, the least significant 5 bits of the MSR are cleared to indicate 100% performance. To enable clock modulation, set bit 4 of this MSR to 1 and write a value from 1-7 in bits 1-3 (where 7 is 87.5% equivalent performance and 1 is 12.5% equivalent performance). More information on clock modulation can be found in volume 3 of the Intel IA64/IA32 Software Developers Manual, under Thermal Monitoring and Protection. Please note that the effect of clock modulation approximates running the CPU at a lower frequency - in benchmarks we have noted up to a 5% variance in performance between clock modulation and running the same core at the equivalent frequency. 

**Quick Quiz 4** : Why would anyone use an Intel system to test out an ARM capability? Answer

Although none of these approaches can be considered a perfect substitute for running on the actual big.LITTLE hardware, they are all likely to be very useful during the time until such hardware is actually available. 

####  What Do You Do With a LITTLE CPU?

If you have both big and LITTLE CPUs, how do you decide what tasks will be banished to the slower LITTLE CPUs? Similarly, if your workload is currently running all on LITTLE CPUs, how do you decide when to take the step of starting up one of the the power-hungry big CPUs? 

Right now for SMP-configured big.LITTLE systems, "you" is the application developer, who can use facilities such as CPU hotplug, affinity, cpusets, sched_mc, and so on to manually direct the available work to the desired subsets of the CPUs. These facilities constrain the scheduler in order to ensure that nothing runs on CPUs that are to be powered down. 

Decisions on what CPUs to use should include a number of considerations. First, if a LITTLE CPU is able to provide sufficient performance, it provides better energy efficiency, at least in cases where race to idle is inappropriate. Second, because mobile platforms have no fans and are sometimes sealed, some devices might not be able to run all the big CPUs at maximum clock rate for very long without overheating. Of course, such devices might also need to limit the heat produced by analog electronics and GPUs as well (see Carroll's and Heiser's 2010 [USENIX paper [PDF]](<http://www.ssrg.nicta.com.au/publications/papers/Carroll_Heiser_10.pdf>) and [presentation [PDF]](<http://static.usenix.org/event/atc10/tech/slides/carroll.pdf>) for a power-consumption analysis of a ca. 2008 smartphone). Third, some workloads can adapt themselves to lower performance. For example, some media applications can reduce performance requirements by dropping frames and reducing resolution. Fourth, there is more to performance than CPU clock speed: For example, it is possible that a workload with high cache-miss rates can run just as fast on a LITTLE CPU as it can on a big CPU. Finally, many workloads will have preferred ways of using the CPUs, for example, some mobile workloads might use the LITTLE CPUs most of the time, but bring the big CPUs online for short bursts of intense processing. 

Keeping track of all this can be challenging, which is one big reason for thinking in terms of automated assistance from the scheduler. Some of the proposed work towards this end is listed in the Future Work and Prospects section. But first, let's take a closer look at CPU hotplug and its potential replacements. 

####  CPU Hotplug: Kill It or Cure It?

Although CPU hotplug has a checkered reputation in many circles, it is what almost all current multicore devices actually use to evacuate work from a given CPU. This is a bit surprising given that CPU hotplug was intended for infrequently removing failing CPUs, not for quickly bringing perfectly good CPUs into and out of service. It is therefore well worth asking what CPU hotplug is providing that users cannot get from other mechanisms: 

  * Migrating timers off of a given CPU. This can likely be fixed, but a synchronous fix that prevents any further timers from being set may be more challenging. 

  * Shutting off a CPU with a single simple action. This can likely be fixed, but requires coordinating interrupts, the scheduler, timers, kthreads, and so on. 

  * Preventing all possible wakeup events from causing that CPU to power back on until the user explicitly permits it to power back on. (Some platforms may have wakeup events that cannot be shut off.) 

  * Synchronous action, so that userspace can treat it atomically. 

  * Coordinating user applications based on hotplug events. (However, there is no known embedded or mobile use of this feature, so if you need it, please let us know. Otherwise it will likely go away.) 

These CPU-hotplug features are valuable outside of the mobile/embedded space, for example, some real-time applications will take a CPU offline and then immediately bring it back online to make it fully available for the application--in particular, to clear timers off of the CPU. Furthermore, people really do make use of CPU hotplug to offline failing CPUs. 

But this brings up another question. Given that CPU hotplug does all these useful things, what is not to like? First, CPU-hotplug operations can take several seconds, as shown [here](<https://wiki.linaro.org/WorkingGroups/PowerManagement/Doc/Hotplug>). An ideal power-management mechanism would have latencies in the 5ms range. It might be possible to make CPU hotplug run much faster. Second, CPU-hotplug offline operations use the `stop_machine()` facility, which interrupts each and every CPU for an extended time period. This sort of behavior is not acceptable when certain types of real-time or high-performance-computing (HPC) applications are running. It might be possible to wean CPU hotplug from `stop_machine()`. 

Third, a given CPU's workqueues can contain large numbers of pending items of work, and migrating all of these can be quite time consuming, as can re-initializing all the workqueue kernel threads when a given CPU comes online. Other CPU-hotplug notifiers have similar problems, which can hopefully be addressed by coming up with a good low-overhead way to "park" and "unpark" kernel threads that are associated with an offline CPU. 

**Quick Quiz 5** : Why not just use `SIGSTOP` to park per-CPU kthreads? Answer

Such a parking mechanism faces the following challenges: 

  * Many per-CPU kernel threads are (quite naturally) coded with the assumption that they will always run on the corresponding CPU. 

  * If a kthread that has an affinity to a given CPU is awakened while that CPU is offline, the scheduler prints an error message and removes the affinity, so that the kthread will now be able to run on any CPU. 

  * Wakeups can be delayed so that they do not arrive at the kthread until after the corresponding CPU has gone offline. 

  * All kernel threads parked for a given offline CPU must sleep interruptibly, because otherwise the kernel will emit soft-lockup messages. 

  * When a given CPU goes offline, any work pending for that CPU must either be completed immediately (thus delaying the offline operation), migrated to some other CPU (thus increasing complexity), or deferred until the CPU comes back online (which might be never). 

**Quick Quiz 6** : What other mechanisms could be used to park per-CPU kthreads? Answer

There is some reason to believe that _any_ mechanism that evacuates all work from a CPU faces these same challenges. 

Finally, CPU-hotplug operations can destroy cpuset configuration, so that cpusets need to be repaired when CPUs are brought back online. This topic is currently the subject of spirited discussions. 

Perhaps these CPU-hotplug shortcomings can be repaired. But suppose that they cannot. What should be done instead in order to evacuate all work from a given CPU? 

Tasks can be moved off of a given CPU by use of explicit per-task affinity, cgroups, or cpusets, although interactions with other uses of these mechanisms need more thought. In addition, interactions among all of these mechanisms can have unexpected results because of a strong desire that the scheduler generally consume less CPU than the workload being scheduled. 

However, interrupts can still happen on a given CPU even after all tasks have been evacuated. Interrupts must be redirected separately using the `/proc/irq` directory. This directory in turn contains one directory for each IRQ, and each IRQ directory contains a `smp_affinity` file to which you can write a hexadecimal mask to restrict delivery of the corresponding interrupts. You can then examine the `/proc/interrupts` file to verify that interrupts really are no longer being delivered to the CPUs in question. See `Documentation/IRQ-affinity.txt` in the kernel sources for more information. One caution: that document notes that some irq controllers do not support affinity, and for such controllers it is not possible to direct irq delivery away from a given CPU. 

Finally, evacuating tasks and interrupts from a given CPU can still leave timers running on that CPU. As noted earlier, there is currently no mechanism other than CPU hotplug to migrate timers off of a given CPU, but it should be possible to create such a mechanism. An asynchronous mechanism (which would allow each timer one final ride on the outgoing CPU) is straightforward. A synchronous mechanism would be more complex, but should be doable. 

So, what should be done? In the near term, the only sane approach is to attack on both fronts: (1) attempt to cure CPU hotplug of its worst ills (especially given that it will likely continue to be needed for removing failing CPUs), and (2) attempt to improve the alternative mechanisms so that they can do more of the work that can currently only be done by CPU hotplug--hopefully avoiding at least some of the complexity currently inherent to CPU hotplug. 

###  Future Work and Prospects

In the short term, the following actions need to be taken: 

  * Create an email list for the attendees and other interested parties. This is now available [here](<http://lists.linaro.org/mailman/listinfo/linaro-sched-sig>), courtesy of Amit Kucheria and Loic Minier. 

  * Document best practices for using existing Linux kernel facilities (including CPU hotplug, cgroups, cpusets, affinity, and so on) to manage big.LITTLE systems in an SMP configuration. This documentation should include measurements of latencies (keeping in mind the 5ms goal for evacuating work from a CPU and for restarting it) and power consumption. Vincent Guittot's [presentation](<https://wiki.linaro.org/WorkingGroups/PowerManagement/Doc/Hotplug>) and [git tree](<http://git.linaro.org/gitweb?p=people/vingu/kernel.git>) are a good start in this direction. 

  * Create software to emulate big.LITTLE systems on current hardware, for example, using one or more of the approaches describe in the  Parallel Hardware/Software Development section. 

  * Produce Linux-kernel-developer-friendly synthetic workloads and benchmarks for mobile applications and use cases, as discussed in the Benchmarks and Synthetic Workloads section. There will be some Linaro work in this direction, but additional workloads and benchmarks are welcome from any and all. 
In the medium term, the following additional actions are needed: 
  * Experiment with improving cpusets and cgroups as discussed in the CPU Hotplug: Kill It or Cure It? section. 

  * Experiment with curing CPU hotplug, also discussed in the CPU Hotplug: Kill It or Cure It? section. 

  * Accumulate a list of the attributes that system-on-a-chip (SoC) vendors believe to be important to scheduling and managing big.LITTLE systems. An initial list was accumulated during the scheduler summit: 

    * Power-domain and clock-domain constraints. For example, many ARM SoCs require that all CPUs in a cluster run at the same clock rate. 

    * Thermal tradeoffs. For example, some SoCs might impose a tradeoff between the number of CPUs running at a given time and the frequency at which they are running if they are to avoid thermal throttling. 

    * Thermal feedback, e.g., temperature sensors. 

    * Process type, where the amount of leakage current can affect the optimal strategies for power-efficient operation. 

    * Relative benefit of reducing frequency of several CPUs as opposed to consolidating workload on a small number of CPUs. 

    * Instruction-per-clock (IPC) measurements and correlation between clock rate and useful forward progress. 

  * Remove `sched_mc`. 
In the longer term, the following additional actions would be quite helpful: 
  * Port Frederic Weisbecker's OS-jitter-reduction patchset to ARM. Geoff Levand of Huawei is leading up an effort along these lines. 

  * Contact gaming companies (e.g., Epic) to see if their 3D gaming engines (which run on both iPhone and Android) can make good use of big.LITTLE, even in the presence of thermal throttling. 

  * Investigate alternative scheduler disciplines. For example, the prototype SCHED_EDF patchset would allow tasks to specify deadlines, which would allow the scheduler to better decide between race-to-idle and run-at-low-frequency. Other related scheduler disciplines such as EVDF might be useful--there may be other real-time technologies that can be commandeered to energy-efficiency purposes. 

If SCHED_EDF looks useful to mobile/embedded, someone needs to forward-port the patch and fix a number of issues in it. This is not a trivial project. (Paul Turner is looking into bringing some Google resources to bear, and Juri Lelli has been doing some recent [deadline-scheduler work](<https://github.com/jlelli/>).) 

One common mobile/embedded requirement is to consolidate the workload down to the minimum number of CPUs that can support acceptable user experience, then spread the load across that minimal set of CPUs. 

  * Investigate modal scheduling. Paul Turner gave the following list of modes as an example: 

    * Low load of interactive, low-utilization tasks might favor race to idle. 

    * Moderate load of periodic media-feeding tasks might lower frequency to the smallest value that allows the task to keep up with its hardware. 

    * High load of CPU-bound tasks in the absence of thermal limitations might increase frequency. 

Some hysteresis will be required. It is usually OK to delay the decisions a bit, especially given that ARM provides relatively fast transitions between power states. Paul Turner posted a first step in this direction, with a [patch series](<https://lkml.org/lkml/2012/2/1/763>) that improves the scheduler's ability to better estimate the effect of migration of each CPU's load. A more up-to-date series is maintained [here](<http://rs5.risingnet.net/~pjt/patches/load_avg/>). 

###  Conclusions

The scheduler mini-summit at Linaro Connect was quite productive, with work already in progress to implement some of the recommendations. For example, some code and patches are in flight to reduce RCU's dependence on `stop_machine()`, which is a first step towards weaning CPU hotplug from `stop_machine()`. For another example, Srivatsa Bhat is doing some good work on curing CPU hotplug of some of its ills. 

So how did we do against the goals? Let's check: 

  1. Take first step towards planning any Linux-kernel scheduler changes that might be needed for ARM's upcoming big.LITTLE systems work well. 

The most important actions toward this goal are the emulation of big.LITTLE systems, the mobile/embedded synthetic benchmarks/workloads, and the list of SoC attributes. This information will help work out which of the longer-term actions are most important. 

  2. Create a power-aware infrastructure for scheduling and related Linux kernel subsystems. For example, integrate dyntick-idle, cpufreq, cpuidle, sched_mc, timers, thermal framework, pm_qos, and the scheduler. 

The mobile/embedded synthetic benchmarks/workloads is the most important first step in this direction, as is the list of SoC attributes. The removal of `sched_mc` is a first implementation step, on the theory that one must tear down before one can build. 

  3. Provide a usable mechanism that reliably allows all work (present and future) to be moved off of a CPU so that that CPU can be powered off and back on under user-application control. CPU hotplug is used for this today, but has some serious side effects, so it would be good to either fix CPU hotplug or come up with a better mechanism--or, in the best Linux-kernel tradition, both. Such a mechanism might also be useful to the real-time people, who also need to clear _all_ non-real-time activity from a given CPU. 

This goal received the most discussion, and the medium-term actions for curing CPU hotplug on the one hand or improving the alternatives to CPU hotplug on the other. 

Work on these three goals has only just begun, but with continued effort, we can make the Linux kernel work better for big.LITTLE in particular and for mobile/embedded workloads on asymmetric systems in general. 

####  Acknowledgments

I am grateful to the scheduler mini-summit attendees for many useful and enlightening discussions, and especially to Amit Kucheria for organizing the mini-summit. We all owe thanks to Zach Pfeffer, Peter Zijlstra, Amit Kucheria, Robin Randhawa, Jason Parker, Rusty Russell, Vincent Guittot, and Dave Rusling for helping make this article human readable. I owe thanks to Dave Rusling and Jim Wasko for their support of this effort. 

####  Answers to Quick Quizzes

**Quick Quiz 1** : But what if there is a different number of Cortex-A7s than of Cortex-A15s? 

**Answer** : In that case, it is necessary to remove the excess CPUs from service, for example, using CPU hotplug, before carrying out the switch. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Why scale down? Isn't it always better to run full out in order to race to idle? 

**Answer** : Although it often is best to race to idle, there are some important exceptions to this rule in certain mobile/embedded workloads. For one example, imagine a codec that required the CPU to occasionally do some work to provide the codec with data. Because CPU power consumption often rises as the square of the core clock frequency, you typically get the best battery life by running the CPU at the lowest frequency that gets the work done in time. As always, use the right tool for the job! 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : I typed the following command: 

```
sudo echo 800000 > /sys/devices/system/cpu/cpu1/cpufreq/scaling_max_freq
```

Despite the `sudo`, I got "Permission denied". Why doesn't `sudo` give me sufficient permissions? 

**Answer** : Although that command does give `echo` sufficient permissions, the actual redirection is carried out by the parent shell process, which evidently does not have sufficient permissions to open the file for writing. One way to work around this is `sudo bash` followed by the `echo`, or to do something like: 

```
sudo sh -c 'echo 800000 > /sys/devices/system/cpu/cpu1/cpufreq/scaling_max_freq'
```

Another approach is to use `tee`, for example: 

```
echo 800000 | sudo tee /sys/devices/system/cpu/cpu1/cpufreq/scaling_max_freq
```

Yet another approach uses `dd` as follows: 

```
echo 800000 | sudo dd of=/sys/devices/system/cpu/cpu1/cpufreq/scaling_max_freq
```

**Back to Quick Quiz 3**.

**Quick Quiz 4** : Why would anyone use an Intel system to test out an ARM capability? 

**Answer** : The scheduler is core code, and for the most part does not care about which instruction-set architecture is running. The important thing is not the ISA, but rather the performance characteristics. 

**Back to Quick Quiz 4**.

**Quick Quiz 5** : Why not just use `SIGSTOP` to park per-CPU kthreads? 

**Answer** : It might well work, at least given appropriate adjustments. Please try it and let us know how it goes. 

**Back to Quick Quiz 5**.

**Quick Quiz 6** : What other mechanisms could be used to park per-CPU kthreads? 

**Answer** : Here are some possibilities: 

  * Kill the kthreads at CPU-offline time and recreate them at CPU-online time. This is used today, and is quite slow. 

  * Kill the kthreads at CPU-offline time and recreate them at CPU-online time, as above, but create a separate `CLONE_` flag to prevent the parent from waiting until the child runs. This waiting behavior exists to work around an old `bash` bug, and is not needed for in-kernel kthreads. 

  * Kill the kthreads at CPU-offline time, but don't recreate them until they are actually needed, perhaps using a separate high-priority kthread to allow the creation to be initiated from environments where blocking is prohibited. This might work well in some situations, but does increase the state space significantly. 

  * Have the kthreads block while their CPU is offline. This approach faces some complications: 

    * Wake-ups can be delayed, so that a delayed wakeup might arrive at a kthread after the corresponding CPU has gone offline. 

    * If a task that has an affinity to a given CPU awakens while that CPU is offline, the scheduler prints a warning message and breaks affinity. This breaking of affinity can cause failures in kthreads written to assume that they only run on their CPU. 

    * Sleeping uninterruptibly while a CPU is offline can result in spurious soft-lockup warnings. 

  * Use an explicit rendezvous to park each kthread, setting its affinity mask to cover all CPUs and informing it that it needs to remain quiescent. This operation would be reversed when the CPU comes back online. This works, but is often surprisingly difficult to get right, particularly on busy systems where wakeups can be delayed. 

  * Your idea here. 

It is quite likely that different approaches will be required in different situations. 

**Back to Quick Quiz 6**.
