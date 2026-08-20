---
title: Improvements in CPU frequency management
url: https://lwn.net/Articles/682391/
date: "April 6, 2016"
category: "Power management-Frequency scaling; Schedutil governor; cpufreq"
author: "April 6, 2016 This article was contributed by Neil Brown"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

April 6, 2016

This article was contributed by Neil Brown

A few years ago, Linux scheduler maintainer Ingo Molnar [expressed](<https://lwn.net/Articles/552885/>) a strong desire that future improvements in CPU power management should integrate well with the scheduler and not try to work independently. Since then, there have been improvements to the way the scheduler estimates load — [per-entity load tracking](<https://lwn.net/Articles/531853/>) in particular — and some [patch sets](<https://lwn.net/Articles/655479/>) circulating that aim to link these improved estimates into the CPU frequency scaling, but the recently released 4.5 kernel still does CPU power management as it has done it for years. This situation appears to be about to change with some preliminary work already [merged](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=277edbabf6fece057b14fb6db5e3a34e00f42f42>) into 4.6-rc1 and some more significant improvements being [prepared](<https://lwn.net/Articles/680876/>) by power management maintainer Rafael Wysocki for likely inclusion in 4.7: the `schedutil` `cpufreq` governor.

The focus of these patch sets is just to begin the process of integration. They don't present a complete solution, but only a couple of steps in the right direction, by creating interfaces for linking the scheduler more closely with CPU power management. 

#### CPU power management in Linux 4.5

Linux support for power management of CPUs roughly divides into `cpuidle`, which guides power usage when the CPU is idle, and `cpufreq`, which governs power usage when it is active. `cpufreq` drivers divide into those, like `intel_pstate` and the Transmeta `longrun`, that choose the CPU frequency themselves using hardware-specific performance counters or firmware functionality, and those that need to be told which power level, from a given range of options, is appropriate at any given time. This power level is nominally a CPU frequency but, as it can be implemented by scaling a high-frequency clock, it is sometimes referred to as a frequency scaling factor. It is not uncommon to scale down the voltage together with the frequency so you can also see terms like DVFS for "Dynamic Voltage and Frequency Scaling", or the more generic OPP for "Operating Power Point". For the most part, we will just refer to frequency changes here.

The `cpufreq` governors that choose a frequency for the second group of drivers are also divided into two groups: the trivial and the non-trivial. There are three trivial frequency governors: `performance`, which always chooses the highest frequency, `powersave`, which always chooses the lowest, and `userspace`, which allows a suitably privileged user-space process to make the decision by writing to a file in `sysfs`. The non-trivial governors vary the CPU frequency based on the apparent system load. This is quite similar to the approach used by `intel_pstate`, except that they determine load from generic information known to the kernel, rather than hardware-specific counters.

The statistics used by these governors do come from the scheduler, but only in a roundabout way that does not reflect the sort of integration that Molnar hopes for. The governors set a timer with a delay of some tens or hundreds of milliseconds and use the CPU-usage statistics (visible in `/proc/stat`) to determine the average load over that time, which is the ratio of busy time for the CPU to total time. There is some subtlety to exactly how this number is used but, roughly, the CPU frequency is increased when the load is above a threshold and decreased if it is below another threshold. The two non-trivial frequency governors differ in their response to increasing load: `on-demand` will immediately jump to the highest frequency and then possibly back off as the idle time increases, making it suitable for interactive tasks, while the `conservative` governor will scale up more gradually as is fitting for background jobs.

#### Problems with the current approach

There are various reasons for dissatisfaction with the current governors. Probably the easiest to identify is the one that caused the Android developers to write their own governor: interactive responsiveness. An idle CPU will likely be at the lowest frequency setting. When a user starts interacting with the device the CPU will continue at that setting for many milliseconds until the timer fires and the new load is assessed. While the `on-demand` governor does go straight to the maximum frequency, it doesn't do so immediately, and the delay is both noticeable and undesirable.

The [`interactive`](<http://mid.gmane.org/1446069953-9701-1-git-send-email-czoborbalint@gmail.com>) governor for Android is designed to detect this transition out of idle and to set a much shorter timeout, typically under ten milliseconds (""1-2 ticks""). A new frequency is then chosen based on the load over this short time range rather than the normal longer interval. The result is improved interactive response without the loss of stability that is likely if all samples were over short time periods.

Another reason for dissatisfaction is that resetting those timers frequently on every CPU is far from elegant. Thomas Gleixner, maintainer of the timer code, is known not to like them and Wysocki [noted](<http://permalink.gmane.org/gmane.linux.power-management.general/72589>) that ""getting rid of those timers allows quite some irritating bugs in cpufreq to be fixed"".

Finally, the information used to guide frequency choice is based entirely on recent history. While it is hard to get reliable information about the future, there is information about the present that can be useful. To understand this it is helpful to consider the different classes of threads as they are seen by the scheduler, which can be divided into realtime, deadline, or normal.

Of these, threads configured for [deadline scheduling](<https://lwn.net/Articles/575497/>) provide the most information about the future. Such threads must specify a worst-case CPU time needed to achieve their goal and a period indicating how often it will be needed. From this, it is possible to calculate the minimum CPU frequency that will allow all deadline threads to be serviced. This can be done without any reference to history.

Realtime threads also provide information about the future, though it is not so detailed. When a realtime thread is ready to run, ""the only possible choice the kernel has"", as Peter Zijlstra [put it](<http://mid.gmane.org/20160301135811.GQ6356@twins.programming.kicks-ass.net>), ""is max OPP"". When there is a realtime thread, but it is currently waiting for something, there are two options. If switching to top speed can be done quickly, then it is safe to ignore the thread until it is runnable, and then instantly switch to maximum frequency as it starts running. If switching the CPU frequency takes a bit longer, then the only really safe thing to do is to stay at maximum CPU frequency whenever there are any realtime threads on that CPU.

Lastly, there are the normal threads, those managed by the [Completely Fair Scheduler](<https://en.wikipedia.org/wiki/Completely_Fair_Scheduler>) (CFS). While the primary source of information available to CFS is historical, it is more fine-grained than the information currently in use. Using per-entity load tracking, CFS knows the recent load characteristics of every thread on each CPU and knows immediately if a thread has exited, a new one has been created, or if a thread has migrated between CPUs. This allows a more precise determination of load, which is already being used for scheduling decisions and could usefully be used for CPU frequency scaling decisions.

CFS can have a little bit more information than historical usage. It is possible for a maximum [CPU bandwidth](<https://lwn.net/Articles/428230/>) to be imposed on a process or process group. CFS could use this to determine an upper limit for the load generated by those processes and may usefully be able to provide that information to `cpufreq`.

#### Challenges

Even if all this information were readily available, and some of it is, making use of it effectively is not necessarily straightforward. This particularly seems to be the case when working in and around the scheduler, as that code is quite performance-sensitive.

One challenge that `cpufreq` must face is how exactly to change the CPU frequency once a decision has been made. As was hinted at above, some platforms may allow fast frequency changes but, while that is true, `cpufreq` doesn't know anything about it. A `cpufreq` driver has a single interface, `target_index()`, to set the frequency, which may take locks or block for other reasons, so it cannot be depended on to be fast. An optional non-blocking interface has been proposed, but it will still be necessary to work with `cpufreq` drivers that need to be called from "process context". This currently means scheduling a worker thread to effect the frequency change.

There is a [question](<http://mid.gmane.org/56CF9D8F.7010607@linaro.org>) whether a regular workqueue thread is really sufficient. It seems possible that on a busy system, such a thread might not get scheduled for a while, so at those times, when a switch to high frequency is most important, it will be most delayed. The `interactive` governor used by Android has addressed this issue by creating a realtime thread to perform all frequency changes. While this works, it does not seem like an ideal solution for the longer term. As noted above, it may be sensible to keep the CPU at maximum speed whenever there are realtime threads. In that case, the existence of a realtime thread for setting the frequency would prevent it from ever being asked to set a lower frequency.

This is currently an open issue. Wysocki is not convinced that this is a real problem in practice, so he is not keen on anything beyond the current simple approach. Those who feel otherwise will need to provide concrete evidence of a problem, which is often a valuable prerequisite for such changes.

#### Two steps forward

There are two patch sets that have been prepared by Wysocki and look likely to reach the mainline soon; one has already been included in 4.6-rc1. There are a number of details in which they differ from others that have been proposed, such as [scheduler-driven frequency selection](<https://lwn.net/Articles/649593/>) originally by Michael Turquette, but possibly the most important is that they are incremental steps with modest goals. They address the issues that can easily be addressed and leave other more difficult issues for further research.

The first of these patch sets removed the timers that Gleixner found so distasteful. Instead of being triggered by a timeout, re-evaluation of the optimal setting for `cpufreq` is now triggered by the scheduler whenever it updates load or runtime statistics. This will doubtless be called more often than frequency changes are really wanted, and possibly more often than it is possible to update the frequency choice, so the `sampling_rate` number that previously set the timer is now used to discard samples until an appropriate interval since the last update has passed.

The new frequency choice is still calculated the same way and it happens about as often, but one noteworthy change is that the updates are no longer synchronized with the scheduler "tick" that timers use. This change has resulted in one measurable [benefit](<http://mid.gmane.org/008201d16458$69b2a4f0$3d17eed0$@net>) for the `intel_pstate` driver, which occasionally made poor decisions due to this synchronization.

The second patch set, some of which is up to its eighth revision, makes two particular changes. A new, optional `fast_switch()` interface is added to `cpufreq` drivers so that fast frequency switching can be used on platforms that support it. If provided, this must be able to run from "interrupt context" meaning that it must be fast and may not sleep. As already discussed, there are times when this can be quite valuable.

The other important change is to introduce a new CPU frequency-scaling governor, `schedutil`, that makes decisions based on utilization as measured by the scheduler. The `cpufreq_update_util()` call that the scheduler makes whenever it updates the load average already carries information about the calculated load on the current CPU, but no governor uses that information. `schedutil` changes that. It doesn't change much though.

`schedutil` still only performs updates at the same rate as the current code, so it doesn't try to address the interactive responsiveness problem, and doesn't try to be clever about realtime or deadline threads. All it does is use the load calculated by the scheduler instead of the average load over the last little while, and optionally imposes that frequency change instantly (directly from the scheduler callback) if the driver supports it.

This is far from a complete solution for power-aware scheduling, but looks like an excellent base on which to make `cpufreq` more responsive to sudden changes in load, and more aware of some of the finer details that the scheduler can, in theory, provide.

It appears that the long-term goal is to get rid of the selectable governors completely and just have a single governor that handles all cases correctly. It would need to respond correctly to realtime tasks, deadline tasks, interactive tasks, and background tasks. These are all concepts that the scheduler must already deal with, so it is quite reasonable to expect that `cpufreq` can learn to deal with them too. It will clearly take a while longer to reach the situation that Molnar desires, but it seems we are well on the way.
