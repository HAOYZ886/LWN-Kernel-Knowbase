---
title: Improving idle behavior in tickless systems
url: https://lwn.net/Articles/775618/
date: "December 28, 2018"
category: "Power management-cpuidle"
author: "December 28, 2018 This article was contributed by Marta Rybczyńska"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

December 28, 2018

This article was contributed by Marta Rybczyńska

Most processors spend a great deal of their time doing nothing, waiting for devices and timer interrupts. In these cases, they can switch to idle modes that shut down parts of their internal circuitry, especially stopping certain clocks. This lowers power consumption significantly and avoids draining device batteries. There are usually a number of idle modes available; the deeper the mode is, the less power the processor needs. The tradeoff is that the cost of switching to and from deeper modes is higher; it takes more time and the content of some caches is also lost. In the Linux kernel, the cpuidle subsystem has the task of predicting which choice will be the most appropriate. Recently, Rafael Wysocki [proposed a new governor](<https://lwn.net/ml/linux-kernel/1994073.q7eAg5jFKm%40aspire.rjw.lan/>) for systems with tickless operation enabled that is expected to be more accurate than the existing menu governor.

#### Cpuidle governors

Predicting the time to the next event is not always an easy task; it is done using a heuristic that depends on the system's recent history. This heuristic can produce incorrect results if the system's behavior changes. Devices cause interrupts at (more or less) predictable intervals that depend on the applications that are running; a cpuidle governor can measure these intervals to make predictions for when the next device interrupt will occur. Also relevant is the regular scheduler timer tick; until a few years ago, kernels always had the timer interrupt running at 100 to 1000 times per second. This picture changed with the introduction of the tickless kernel; periods without interrupts can be longer (as the timer tick may be disabled) and, as a result, the processor can possibly enter deeper idle states.

Linux currently provides two cpuidle governors that [reside in `drivers/cpuidle/governors`](<https://elixir.bootlin.com/linux/v4.20-rc6/source/drivers/cpuidle/governors>); they are called "ladder" and "menu". The basic ideas and interfaces of the cpuidle governors were [discussed here back in 2010](<https://lwn.net/Articles/384146/>). The ladder governor chooses the shallowest idle mode first and then moves to the next deeper mode if the observed wait time is long enough. It is considered to be the better choice when running a system with regular clock ticks and when power consumption is not an important factor. The disadvantage of the ladder governor is that it may need a long time to reach a deep idle mode. The menu governor is, until now, the preferred choice for tickless systems. It tries to choose the most appropriate idle mode, not necessarily a shallow one. The user can check the governor they are running in `/sys/devices/system/cpu/cpuidle/current_governor_ro`. 

#### The critique of the menu governor

The menu governor tries to find the deepest idle state that can be entered in the given conditions. It predicts the duration of the next idle period based on past history, then it correlates the observed recent idle durations with the idle states available to choose the idle state that will most likely match with the next idle interval to come. The menu governor applies different corrective factors for the time until the next predicted wakeup, including the system load and the number of tasks waiting for I/O. The corrective factors have, as their goal, limiting the performance impact of entering the idle states.

Wysocki noticed multiple issues that, according to him, make the menu governor not as accurate as it should be. The first observation is related to the creation of the interrupt history pattern. The menu governor uses all interrupts, including timers, to predict when the next one will happen. On the other hand, it already has the information when the next timer tick will happen, but does not correlate the two. As a result, it may happen that the governor predicts a wakeup (that would be a timer) when it should already know that the next timer event will actually happen later.

The second observation is that the governor uses the number of processes waiting for I/O as a corrective factor. The reason for this was the desire to lower the impact of idle modes for highly loaded systems. Entering deeper idle modes on such systems may have a more visible impact on performance, so the correction steers the governor toward the shallower modes. According to Wysocki, the number of processes waiting for I/O has no impact on the idle modes available, and should not be taken into account. Finally, he argued that the pattern detection used by the menu governor sometimes considers values that are too large to matter in practice. Those values could be omitted and the analysis would then use less resources.

Wysocki was considering a rework of the menu governor to address these issues, but that could worsen the performance of workloads that are tuned to work well with the current implementation of the menu governor. Because of that, he chose to implement a new governor, allowing users to benchmark the impact of the two in their actual workloads and make their own choice.

#### The timer events oriented governor

The new governor is called the "timer events oriented" (TEO) governor. It uses the same strategy as the menu governor — predicting the next idle duration and then selecting the idle mode that fits best — but the factors it takes into consideration are different. The concept behind TEO is that the most frequent source of CPU wakeups on many systems is timer events, not device interrupts. Wysocki notes that timer events might be even two or more orders of magnitude more frequent than other interrupts. So the time until the next timer event alone provides a strong predictive clue. 

Another observation is that it is enough to use the recent past to provide accurate estimations of idle periods. In systems where wakeup sources other than timers are more important, this observation does not apply directly. Still, Wysocki argues, the analysis can be based only on a few idle-time intervals. In particular, only intervals that are shorter than the time to the next timer event need to be considered. This is because the longer durations are likely to belong to patterns that can be approximated to the closest timer, anyway.

TEO is designed around the idea that it is likely that the next wakeup will be the expiration of the next timer event; it chooses the deepest idle mode that corresponds to this interval. Then it verifies if this interval also matches the non-timer events, as seen in the pattern of observed idle times from the recent past. If the idle mode selected matches both the timer and non-timer events, it becomes the final choice; otherwise, TEO tries again with a shallower mode. 

The algorithm also covers the case when the pattern is changing; there is a special check to determine whether most of the recent idle durations were too short for the idle mode selected. If this is the case, then TEO uses only those values to calculate the new expected idle duration. Then it selects the idle state again, which will result in selecting a shallower one.

#### Current state and benchmark results

The patch is in its [tenth version](<https://lwn.net/ml/linux-kernel/2830755.AcCr0sspTm%40aspire.rjw.lan/>) at the time of this writing. Different developers have started evaluating the code. Giovanni Gherdovich [shared benchmark results](<https://lwn.net/ml/linux-pm/1541877001.17878.5.camel@suse.cz/>) from the patch; they show a number of cases when the choice of the cpuidle governor has no importance and others where TEO usually offers a slight improvement in performance compared to menu. The [detailed results](<https://beta.suse.com/private/ggherdovich/teo-eval/teo-v6-eval.html>) are available separately for different versions of the patch, illustrating the impact on bandwidth and I/O latency. Doug Smythies provided other benchmark results and [reported that performance improves and power usage stays the same](<https://lwn.net/ml/linux-kernel/000001d495ab$5c16eea0$1444cbe0$@net/>).

The TEO governor is in an early stage. As the code is subtle, it will still require more work and benchmarking in different systems and architectures, especially with regard to the impact on the power consumption. In addition, Wysocki has also been working on other aspects of the power consumption and idle modes, [presenting the work at Kernel Recipes](<https://lwn.net/Articles/767630/>). The early results are encouraging. The goal of the development -- better prediction of the next idle mode to use -- seems to be reached.
