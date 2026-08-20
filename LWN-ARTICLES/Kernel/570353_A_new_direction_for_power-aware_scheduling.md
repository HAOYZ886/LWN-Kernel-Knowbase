---
title: "A new direction for power-aware scheduling"
url: https://lwn.net/Articles/570353/
date: "October 15, 2013"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet October 15, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
October 15, 2013

Power-aware scheduling attempts to place processes on CPUs in a way that minimizes the system's overall consumption of power. Discussion in this area has been muted since we [last looked at it](<https://lwn.net/Articles/554956/>) in June, but work has been proceeding. Now a new set of power-aware scheduling patches shows a significant change in direction motivated by criticisms that were aired in June. This particular problem is far from solved, but the shape of the eventual solution may be becoming a bit more clear. 

Thus far, most of the power-aware scheduling patches posted to the lists have been focused on task placement — packing "small" processes onto a small number of CPUs to allow others to be powered down, for example. The problem with that approach, as Ingo Molnar [complained](<https://lwn.net/Articles/552885/>) at the time, was that it failed to recognize that there are several mechanisms used to control CPU power consumption. These include the cpuidle subsystem (which decides when a CPU can sleep and how deeply), the cpufreq subsystem (charged with controlling the clock frequency for CPUs) and various aspects of the scheduler itself. There is no integration between these subsystems; indeed, the scheduler is almost entirely ignorant of what the cpuidle and cpufreq controllers are doing. There are other problems as well: the notion of controlling a CPU's frequency has been effectively rendered obsolete by current processor designs, for example. 

In the end, Ingo said that no power-aware scheduling patches would be considered for merging until these problems were solved. In other words, the developers working on these patches needed to solve not just their problem, but the problem of rationalizing and integrating the work that has been done by other developers in preceding years. Such things happen in kernel development; it can be hard on individual developers, but it does result in better code in the long term. 

#### The latest approach

To address this challenge, Morten Rasmussen, who has been working on the [big LITTLE MP scheduler](<https://lwn.net/Articles/541005/>), has taken a step back; his latest [power-aware scheduling patch set](<https://lwn.net/Articles/570162/>) does not actually introduce much in the way of power-aware scheduling. Instead, it is focused on the creation of an internal API that governs communications between the scheduler and a new type of "power driver" that is meant to eventually replace the cpuidle and cpufreq subsystems. The power driver (there can only be one for all CPUs in the current patch set) provides these operations to the scheduler: 

```
struct power_driver {
    	int (*at_max_capacity)	(int cpu);
    	int (*go_faster)	(int cpu, int hint);
    	int (*go_slower)	(int cpu, int hint);
    	int (*best_wake_cpu)	(void);
    	void (*late_callback)	(int cpu);
        };
```

Two of these methods allow the scheduler to query the power state of a given CPU; `at_max_capacity()` allows the scheduler to ask whether the processor is running at full speed, while `best_wake_cpu()` asks which (sleeping) CPU would be the best to wake in response to increasing load. The `best_wake_cpu()` call can make use of low-level architectural knowledge to determine which CPU would require the least power to bring up; it would, for example, favor CPUs that share power or clock lines with currently running CPUs over those that would require powering up a new package. 

The scheduler can provide feedback to the power driver with the `go_faster()` and `go_slower()` methods. These calls request higher or lower speed from the given CPU without specifying an actual clock frequency, which isn't really possible on a lot of current processors. The power driver can then instruct the hardware to adopt a power policy that matches what the scheduler is asking for. The `hint` parameter is not used in the current patch set; its purpose is to indicate how much faster or slower performance the scheduler would like to see. These calls as a whole are hints, actually; the power driver is not required to carry out the scheduler's wishes. 

Finally, `late_callback()` exists to allow the power driver to do work that may require sleeping or having interrupts enabled. Most of the functions listed above can be called from within the scheduler at almost any point, so they have to be written to run in atomic context. If they need to do something that cannot be done in that context, they can set the work aside; the scheduler will call `late_callback()` at a safe time for that work to be done. 

The current patch set makes just enough use of these functions to show how they would be used. Whenever the scheduler adds a process to a given CPU's run queue, it checks whether the total load exceeds what the CPU is able to provide; if so, a call to `go_faster()` will be made to ask for more performance. A similar test is done whenever a process is removed from a CPU; if that CPU is providing more power than is needed, `go_slower()` will be called. A separate test will call `go_faster()` if the idle time on the CPU is low, even if the computed load suggests that the CPU should not be busy. Rudimentary implementations of `go_faster()` and `go_slower()` have been provided; they are a simple wrapper around the existing cpufreq driver code. 

#### What's coming

The full plan (as described in [Morten's Linux Plumbers Conference talk slides [PDF]](<http://www.linuxplumbersconf.org/2013/ocw//system/presentations/1263/original/Unifying_Power_Policies_LPC.pdf>)) calls for the elimination of cpufreq and cpuidle altogether once their functionality has been pulled into the power driver. There will also be several more functions to be provided by the power driver. These include `get_best_sleep_cpu()` to get a hint for the best CPU to put asleep, `enter_idle()` to actually put a CPU into the sleep state, `load_scale()` to help with the calculation of loads regardless of the CPU's current power state, and `task_boost()` to give priority to a specific CPU. `task_boost()` is aimed at systems that provide features like "turbo mode," where one CPU can be overclocked, but only if the others are idle. 

The long-term plan also involves bringing back techniques like [small-task packing](<https://lwn.net/Articles/520857/>), proper support for big.LITTLE systems, and more. But those goals look distant at the moment; Morten and company must first build a consensus around the proposed architecture. That may take some doing, yet; scheduler developer Peter Zijlstra's [first response](<https://lwn.net/Articles/570374/>) was ""I don't see anything except a random bunch of hooks without an over-all picture of how to get less power used."" Morten has promised to fill out the story. 

Some of these issues may be resolved on October 23, when a [half-day minisummit](<https://lwn.net/Articles/570375/>) will be held on the topic in Edinburgh. Many of the relevant developers should be there, allowing for quick resolution of a number of the outstanding issues. With luck, your editor will be there too; stay tuned for the next episode in this long-running story.
