---
title: "Frequency-invariant utilization tracking for x86"
url: https://lwn.net/Articles/816388/
date: "April 2, 2020"
category: "Power management-Frequency scaling; Scheduler-Load tracking; Schedutil governor"
author: "By Jonathan Corbet April 2, 2020"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 2, 2020

The kernel provides a number of CPU-frequency governors to choose from; by most accounts, the most effective of those is "schedutil", which was [merged](<https://git.kernel.org/linus/9bdcb44e391d>) for the 4.7 kernel in 2016. While schedutil is used on mobile devices, it still doesn't see much use on x86 desktops; the [intel_pstate](<https://www.kernel.org/doc/html/v4.12/admin-guide/pm/intel_pstate.html>) governor is generally seen giving better results on those processors as a result of the secret knowledge embodied therein. A set of patches merged for 5.7, though, gives schedutil a better idea of what the true utilization of x86 processors is and, as a result, greatly improves its effectiveness. 

Appropriate CPU-frequency selection is important for a couple of reasons. If a CPU's frequency is set too high, it will consume more power than needed, which is a concern regardless of whether that CPU is running in a phone or a data center. Setting the frequency too low, instead, will hurt the performance of the system; in the worst cases, it could cause the available CPU power to be inadequate for the workload and, perhaps, even increase power consumption by keeping system components awake for longer than necessary. So there are plenty of incentives to get this decision right. 

One key input into any CPU-frequency algorithm is the amount of load a given CPU is experiencing. A heavily loaded processor must, naturally, be run at a higher frequency than one which is almost idle. "Load" is generally characterized as the percentage of time that a CPU is actually running the workload; a CPU that is running flat-out is 100% loaded. There is one little detail that should be taken into account, though: the current operating frequency of the CPU. A CPU may be running 100% of the time, but if it is at 50% of its maximum frequency, it is not actually 100% loaded. To deal with this, the kernel's load tracking scales the observed load by the frequency the CPU is running at; this scaled value is used to determine how loaded a CPU truly is and how its frequency should change, if at all. 

At least, that's how it is done on some processors. On x86 processors, though, this frequency-invariant load tracking isn't available; that means that frequency governors like schedutil cannot make the best decisions. It is not entirely surprising that performance (as measured in both power consumption and CPU throughput) suffers. 

This would seem like an obvious problem to fix. The catch is that, on contemporary Intel processors, it is not actually possible to know the operating frequency of a CPU. The operating system has some broad control over the operating power point of the CPU and can make polite suggestions as to what it should be, but details like actual running frequency are dealt with deep inside the processor package and the kernel is not supposed to worry its pretty little head about them. Without that information, it's not possible to get the true utilization of an x86 processor. 

It turns out that there is a way to approximate this information, though; it was [suggested](<https://lore.kernel.org/lkml/52f711be59539723358bea1aa3c368910a68b46d.1459485198.git.len.brown@intel.com/>) by Len Brown back in 2016 but not adopted at that time. There are two [model-specific registers (MSRs)](<https://en.wikipedia.org/wiki/Model-specific_register>) on modern x86 CPUs called `APERF` and `MPERF`. Both can be thought of as a sort of time-stamp counter that increments as the CPU executes (though Intel stresses that the contents of those registers don't have any specific timekeeping semantics). `MPERF` increments at constant a rate proportional to the maximum frequency of the processor, while `APERF` increments at a variable rate proportional to the actual operating frequency. If `aperf_change` is the change in `APERF` over a given time period, and `mperf_change` is the change in `MPERF` over that same period, then the operating frequency can be approximated as: 

```
operating_freq = (max_freq*aperf_change)/mperf_change;
```

Reading those MSRs is relatively expensive, so this calculation cannot be made often, but once per clock tick (every 1-10ms) turns out to be enough. 

There is one other little detail, though, in the form of Intel's "turbo mode". Old timers will be thinking of [the button](<https://en.wikipedia.org/wiki/Turbo_button>) on the case of a PC that would let it run at a breathtaking 6MHz, but this is different. When the load in a particular package is concentrated on a small number of CPUs, and the others are idle, the busy CPUs can be run at a frequency higher than the alleged maximum. That makes it hard to know what the true utilization of a CPU is, because its capacity will vary depending on what other CPUs in the system are doing. 

[The patches](<https://lwn.net/ml/linux-kernel/20200122151617.531-1-ggherdovich@suse.cz/>) (posted by Giovanni Gherdovich) implement the above mentioned method to calculate the operating frequency, and use the turbo frequency attainable by four processors simultaneously as the maximum possible. The result is a reasonable measure of what the utilization of a given processor is. That lets schedutil make better decisions about what the operating frequency of each CPU should actually be. 

As it happens, the algorithm used by schedutil to choose a frequency changes a bit when it knows that the utilization numbers it gets are frequency-invariant. Without invariance, schedutil will move frequencies up or down one step at a time. With invariance, it can better calculate what the frequency should be, so it can make multi-step changes. That allows it to respond more quickly to the actual load. 

The end result, Gherdovich said in the patch changelog, is performance from schedutil that is ""closer to, if not on-par with, the powersave governor from the intel_pstate driver/framework"". To back that up, the changelog includes a long series of benchmark results; the changelog is longer than the patch itself. While the effects of the change are not all positive, the improvements (in both performance and power usage) tend to be large while the regressions happen with more focused benchmarks and are relatively small. One of the workloads that benefits the most is kernel compilation, a result that almost guarantees acceptance of the change in its own right. 

The curious can read the patch changelog for the benchmark results in their full detail. For the rest of us, what really matters is that the schedutil CPU-frequency governor should perform much better on x86 machines than it has in the past. Whether that will cause distributions to switch to schedutil remains to be seen; that will depend on how well it works on real-world workloads, which often have a disturbing tendency to not behave the way the benchmarks did.
