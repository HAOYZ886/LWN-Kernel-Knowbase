---
title: Telling the scheduler about thermal pressure
url: https://lwn.net/Articles/788380/
date: "May 16, 2019"
category: Scheduler; Thermal management
author: "May 16, 2019 This article was contributed by Marta Rybczyńska"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

May 16, 2019

This article was contributed by Marta Rybczyńska

Even with radiators and fans, a system's CPUs can overheat. When that happens, the kernel's thermal governor will cap the maximum frequency of that CPU to allow it to cool. The scheduler, however, is not aware that the CPU's capacity has changed; it may schedule more work than optimal in the current conditions, leading to a performance degradation. Recently, Thara Gopinath did some research and [posted a patch set](<https://lwn.net/ml/linux-kernel/1555443521-579-1-git-send-email-thara.gopinath@linaro.org/>) to address this problem. The solution adds an interface to inform the scheduler about thermal events so that it can assign tasks better and thus improve the overall system performance.

The thermal framework in Linux includes a number of elements, including the thermal governor. Its task is to manage the temperature of the system's thermal zones, keeping it within an acceptable range while maintaining good performance (an overview of the thermal framework can be found in [this slide set [PDF]](<https://blog.linuxplumbersconf.org/2015/ocw//system/presentations/2613/original/thermal-framework-status-no-transitioning.pdf>)). There are a number of thermal governors that can be found in the [`drivers/thermal/` subdirectory of the kernel tree](<https://elixir.bootlin.com/linux/v5.1/source/drivers/thermal>). If the CPU overheats, the governor may cap the maximum frequency of that CPU, meaning that the processing capacity of the CPU gets reduced too.

The CPU capacity in the scheduler is a value representing the ability of a specific CPU to process tasks (interested readers can find more information in [this article](<https://lwn.net/Articles/639543/>)). The capacities of the CPUs in a system may vary, especially on architectures like big.LITTLE. The scheduler knows (at least it assumes it knows) how much work can be done on each CPU; it uses that information to balance the task load across the system. If the information the scheduler has on what a given CPU can do is inaccurate because of thermal events (or any other frequency capping), it is likely to put too much work onto that CPU.

Gopinath introduces a term that is useful when talking about this kind of event: "thermal pressure", which is the difference between the maximum processing capacity of a CPU and the currently available capacity, which may be reduced by overheating events. Gopinath explained in the patch set cover letter that the raw thermal pressure is hard to observe and that there is a delay between the capping of the frequency and the scheduler taking it into account. Because of this, the proposal is to use a weighted average over time, where the weight corresponds to the amount of time the maximum frequency was capped. 

#### Different algorithms and their benchmarks

Gopinath tried multiple algorithms while working on this project (an [earlier version](<https://lwn.net/ml/linux-kernel/1539102302-9057-1-git-send-email-thara.gopinath@linaro.org/>) of the patch set was posted in October 2018) and presented a comparison with benchmark results.

The first idea was to directly use the instantaneous value of the capped frequency in the scheduler; this algorithm improved performance, but only slightly. The other two algorithms studied use a weighted average. The first of those reused the [per-entity load tracking](<https://lwn.net/Articles/531853/>) (PELT) algorithm that is used to track the CPU load created by processes (and control groups); this variant incorporates averages of the realtime and deadline load and utilization. The final approach just uses a simple decay-based metric for thermal pressure, with a variable decay period. Both weighted-average algorithms gave better results than the instantaneous value, with throughput improvements on the order of 3-4%. The non-PELT version performed slightly better. 

Ingo Molnar [reviewed the results](<https://lwn.net/ml/linux-kernel/20190417055514.GA27400@gmail.com/>) and responded positively to the framework, but would like to see more benchmarks run. He suggested testing more decay periods. Gopinath [agreed](<https://lwn.net/ml/linux-kernel/5CB75FD9.3070207@linaro.org/>), saying that tests on different system-on-chips (SoCs) would be a good idea, as the best decay period could differ between the systems. In addition, a configurable decay period is something that is planned.

In parallel, Peter Zijlstra [noted](<https://lwn.net/ml/linux-kernel/20190424163424.GG4038@hirez.programming.kicks-ass.net/>) that he would prefer a PELT-based approach instead of mixing different averaging algorithms. Molnar [dug into](<https://lwn.net/ml/linux-kernel/20190425173333.GA4081@gmail.com/>) the PELT code for ways to obtain better results with the existing algorithm. He found that the decay is set to a constant; on the other hand Gopinath's work shows that the performance depends heavily on its value. It should be possible to get better results with PELT if the code can be suitably modified. It [looks like](<https://lwn.net/ml/linux-kernel/CAKfTPtBQs9yBPQbzZykvEca-pjjuSmB2wTAHJbFxuW-ew-ew7A@mail.gmail.com/>) at least one solution has been found that doesn't require significant changes.

Ionela Voinescu [ran some benchmarks](<https://lwn.net/ml/linux-kernel/632321a8-d7f0-49a6-9577-95fac4c87b1c@arm.com/>) in different conditions and found that the thermal pressure is indeed useful, but without a clear conclusion on which averaging algorithm to use. Gopinath and Voinescu agreed that more benchmarking will be needed.

#### The thermal pressure API

Gopinath's work introduces an API that allows the scheduler to be notified about thermal events. It includes two new functions. The first, `sched_update_thermal_pressure()`, should be called by any module that caps the maximum CPU frequency; its prototype is:

```
void sched_update_thermal_pressure(struct cpumask *cpus,
                                           unsigned long cap_max_freq,
                                           unsigned long max_freq);
```

The mask of the CPUs to update the thermal pressure is passed in `cpus`, the new (capped) maximum frequency in `cap_max_freq`, and the available maximum frequency without any thermal events is in `max_freq`.

The scheduler can also obtain the thermal pressure of a given CPU by calling:

```
unsigned long sched_get_thermal_pressure(int cpu);
```

Internally, the thermal pressure framework uses a per-CPU `thermal_pressure` structure to keep track of the current and old values of the thermal pressure along with the time it was last updated. Currently, the update happens from a periodic timer. However, during the discussion, Quentin Perret [suggested](<https://lwn.net/ml/linux-kernel/20190425105658.q45cmfogrt6wwtih@queper01-ThinkPad-T460s/>) that it be updated at the same time as other statistics. Doing this work during the load-balancing statistics update was proposed first, but Perret later [suggested](<https://lwn.net/ml/linux-kernel/20190508090547.4glnypolmiw3cun4@queper01-lin/>) that the thermal-statistics update would be a better time; that would allow shorter decay periods and more accuracy for low-latency tasks.

The developers [discussed](<https://lwn.net/ml/linux-kernel/20190418094833.owlobrx6x5gclvhy@queper01-lin/>) whether user-space frequency capping should be included in the framework. The user (or a user-space thermal daemon) might change the maximum frequency for thermal reasons. On the other hand, that capping will last for seconds or more — which is different than capping by the thermal framework — and the reason for the change may be something other than thermal concerns. Whether the thermal pressure framework will include frequency capping from user space remains an open question for now. 

[Molnar asked](<https://lwn.net/ml/linux-kernel/20190417182932.GB5140@gmail.com/>) whether there is a connection between the thermal pressure approach and [energy-aware scheduling](<https://lwn.net/Articles/749900/>) (EAS). Gopinath [replied](<https://lwn.net/ml/linux-kernel/5CB7BFBC.9090007@linaro.org/>) that the two approaches have different scope: thermal pressure is going to work better in asymmetric configurations where capacities are different and it is more likely to cause the scheduler to move tasks between CPUs. The two approaches should also be independent because thermal pressure should work even if EAS is not compiled in.

#### Current status and next steps

The kernel developers seem receptive to the proposed idea. It is likely that this, or a similar, framework will be merged in the future. Before that happens, there is still some work left: figuring out the details of the algorithm to be included (and whether to reuse the PELT code), the details of the decay period, and, of course, more benchmarking in different systems. Interested readers can find the Gopinath's [slides from the Linux Plumbers Conference [PDF]](<https://www.linuxplumbersconf.org/event/2/contributions/183/attachments/41/48/Thermal_Pressure__Scheduler-Thermal_Interactions-Thara.pdf>) that offer additional background information for the earlier version of the work.
