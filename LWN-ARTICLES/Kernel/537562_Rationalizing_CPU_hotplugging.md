---
title: Rationalizing CPU hotplugging
url: https://lwn.net/Articles/537562/
date: "February 12, 2013"
category: Hotplug; Notifiers
author: "By Jonathan Corbet February 12, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 12, 2013

One of the leading sources of code churn in the 3.8 development cycle was the removal of the `__devinit` family of macros. These macros marked code and data that were only needed during device initialization and which, thus, could be disposed of once initialization was complete. These macros are being removed for a simple reason: hardware has become so dynamic that initialization is _never_ complete; something new can always show up, and there is no longer any point in building a kernel that cannot cope with transient devices. Even in this world, though, CPUs are generally seen as being static. But CPUs, too, can come and go, and that is motivating changes in how the kernel manages them. 

Hotplugging is a familiar concept when one thinks about keyboards, printers, or storage devices, but it is a bit less so for CPUs: USB-attached add-on processors are still relatively rare in the market. Even so, the kernel has had support for CPU hotplug for some time; the original version of [Documentation/cpu-hotplug.txt](<https://lwn.net/Articles/537570/>) was added in 2006 for the 2.6.16 kernel. That document mentioned a couple of use cases for this feature: high-end NUMA hardware that truly has runtime-pluggable processors, and the ability to disable a faulty CPU in a high-reliability system. Other uses have since come along, including system suspend operations (where all CPUs but one are "unplugged" prior to suspending the system) and virtualization, where virtual CPUs can be given to (or taken from) guests at will. 

So CPU hotplug is a useful feature, but the current implementation in the kernel is not well loved; in a recent [patch set](<https://lwn.net/Articles/535764/>) intended to improve the situation, Thomas Gleixner remarked that ""the current CPU hotplug implementation has become an increasing nightmare full of races and undocumented behaviour."" CPU hotplug shows a lot of the signs of a feature that has evolved significantly over time without high-level oversight; among other things, the sequence of steps followed for an unplug operation is not the reverse of the steps to plug in a new CPU. But much of the trouble associated with CPU hotplug is blamed on its extensive use of notifiers. 

The kernel's [notifier](<https://lwn.net/Articles/185500/>) mechanism is a way for kernel code to request a callback when an event of interest happens. They are, in a sense, general-purpose hooks that anybody in the kernel can use — and, it seems, just about anybody does. There have been a lot of complaints about notifiers, as is typified by [this comment from Linus](<https://lwn.net/Articles/537577/>) in response to Thomas's patch set: 

Notifiers are a disgrace, and almost all of them are a major design mistake. They all have locking problems, [they] introduce internal arbitrary API's that are hard to fix later (because you have random people who decided to hook into them, which is the whole *point* of those notifier chains). 

Notifiers also make the code hard to understand because there is no easy way to know what will happen when a notifier chain (which is a run-time construct) is invoked: there could be an arbitrary set of notifiers in the chain, in any order. The ordering requirements of specific notifiers can add some fun challenges of their own. 

The process of unplugging a CPU requires a surprisingly long list of actions. The scheduler must be informed so it can migrate processes off the affected CPU and shut down the relevant run queue. Per-CPU kernel threads need to be told to exit or "park" themselves. CPU frequency governors need to be told to stop worrying about that processor. Almost anything with per-CPU variables will need to make arrangements for one CPU to go away. Timers running on the outgoing CPU need to be relocated. The read-copy-update subsystem must be told to stop tracking the CPU and to ensure that any RCU callbacks for that CPU get taken care of. Every architecture has its own low-level details to take care of. The perf events subsystem has an impressive set of requirements of its own. And so on; this list is nowhere near comprehensive. 

All of these actions are currently accomplished by way of a set of notifier callbacks which, with luck, get called in the right order. Meanwhile, plugging in a new CPU requires an analogous set of operations, but those are handled in an asymmetric manner with a different set of callbacks. The end result is that the mechanism is fragile and that few people have any real understanding of all the steps needed to plug or unplug a CPU. 

Thomas's objective is not to rewrite all those notifier functions or fundamentally change what is done to implement a CPU hotplug operation — at least, not yet. Instead, he is focused on imposing some order on the whole process so that it can be understood by looking at the code. To that end, he has replaced the current set of notifier chains with a linear sequence of states to be worked through when bringing up or shutting down a CPU. There is a single array of `cpuhp_step` structures, one per state: 

```
struct cpuhp_step {
    	int (*startup)(unsigned int cpu);
    	int (*teardown)(unsigned int cpu);
        };
```

The `startup()` function will be called when passing through the state as a new CPU is brought online, while `teardown()` is called when things are moving in the other direction. Many states only have one function or the other in the current implementation; the eventual goal is to make the process more symmetrical. In the initial patch set, the set of states is: 

> State| startup| teardown |   
> ---|---|---|---  
> `CPUHP_CREATE_THREADS` | ✔ | |   
> `CPUHP_PERF_X86_UNCORE_PREP` | ✔ | ✔|   
> `CPUHP_PERF_X86_PREPARE` | ✔ | ✔|   
> `CPUHP_PERF_BFIN` | ✔ | |   
> `CPUHP_PERF_POWER` | ✔ | |   
> `CPUHP_PERF_SUPERH` | ✔ | |   
> `CPUHP_PERF_PREPARE` | ✔ | ✔|   
> `CPUHP_SCHED_MIGRATE_PREP` | ✔ | ✔|   
> `CPUHP_WORKQUEUE_PREP` | ✔ | |   
> `CPUHP_RCUTREE_PREPARE` | ✔ | ✔|   
> `CPUHP_HRTIMERS_PREPARE` | ✔ | ✔|   
> `CPUHP_TIMERS_PREPARE` | ✔ | ✔|   
> `CPUHP_PROFILE_PREPARE` | ✔ | ✔|   
> `CPUHP_X2APIC_PREPARE` | ✔ | ✔|   
> `CPUHP_SMPCFD_PREPARE` | ✔ | ✔|   
> `CPUHP_SMPCFD_PREPARE` | ✔ | |   
> `CPUHP_SLAB_PREPARE` | ✔ | ✔|   
> `CPUHP_NOTIFY_PREPARE` | ✔ | |   
> `CPUHP_NOTIFY_DEAD` |  | ✔|   
> `CPUHP_CPUFREQ_DEAD` |  | ✔|   
> `CPUHP_SCHED_DEAD` |  | ✔|   
> `CPUHP_CLOCKEVENTS_DEAD` |  | ✔|   
> `CPUHP_BRINGUP_CPU` | ✔ | |   
> `CPUHP_AP_OFFLINE` |  | | Application processor states  
> `CPUHP_AP_SCHED_STARTING` | ✔ | |   
> `CPUHP_AP_PERF_X86_UNCORE_STARTING` | ✔ | |   
> `CPUHP_AP_PERF_X86_AMD_IBS_STARTING` | ✔ | ✔|   
> `CPUHP_AP_PERF_X86_STARTING` | ✔ | ✔|   
> `CPUHP_AP_PERF_ARM_STARTING` | ✔ | |   
> `CPUHP_AP_ARM_VFP_STARTING` | ✔ | ✔|   
> `CPUHP_AP_ARM64_TIMER_STARTING` | ✔ | ✔|   
> `CPUHP_AP_KVM_STARTING` | ✔ | ✔|   
> `CPUHP_AP_X86_TBOOT_DYING` |  | ✔|   
> `CPUHP_AP_S390_VTIME_DYING` |  | ✔|   
> `CPUHP_AP_CLOCKEVENTS_DYING` |  | ✔|   
> `CPUHP_AP_RCUTREE_DYING` |  | ✔|   
> `CPUHP_AP_SCHED_NOHZ_DYING` |  | ✔|   
> `CPUHP_AP_SCHED_MIGRATE_DYING` |  | ✔|   
> `CPUHP_AP_MAZ` |  | | End marker for AP states  
> `CPUHP_TEARDOWN_CPU` |  | ✔|   
> `CPUHP_PERCPU_THREADS` | ✔ | ✔|   
> `CPUHP_SCHED_ONLINE` | ✔ | ✔|   
> `CPUHP_PERF_ONLINE` | ✔ | ✔|   
> `CPUHP_SCHED_MIGRATE_ONLINE` | ✔ | |   
> `CPUHP_WORKQUEUE_ONLINE` | ✔ | ✔|   
> `CPUHP_CPUFREQ_ONLINE` | ✔ | ✔|   
> `CPUHP_RCUTREE_ONLINE` | ✔ | ✔|   
> `CPUHP_NOTIFY_ONLINE` | ✔ | |   
> `CPUHP_PROFILE_ONLINE` | ✔ | |   
> `CPUHP_SLAB_ONLINE` | ✔ | ✔|   
> `CPUHP_NOTIFY_DOWN_PREPARE` |  | ✔|   
> `CPUHP_PERF_X86_UNCORE_ONLINE` | ✔ | ✔|   
> `CPUHP_PERF_X86_ONLINE` | ✔ | |   
> `CPUHP_PERF_S390_ONLINE` | ✔ | ✔|   
  
Looking at that list, one begins to see why the current CPU hotplug mechanism is hard to understand. Things are messy enough that Thomas is not really trying to change anything fundamental in how CPU hotplug works; most of the existing notifier callbacks are still there, they are just invoked in a different way. The purpose of the exercise, [Thomas said](<https://lwn.net/Articles/537608/>), was: 

It's about making the ordering constraints clear. It's about documenting the existing horror in a way, that one can understand the hotplug process w/o hallucinogenic drugs. 

Once some high-level order has been brought to the CPU hotplug mechanism, one can think about trying to clean things up. The eventual goal is to have a much smaller set of externally visible states; for drivers and filesystems, there will only be "prepare" and "enable" states available, with no ordering between subsystems. Also, notably, drivers and filesystems will not be allowed to cause a hotplug operation (in either direction) to fail. When the process is complete, the hotplug subsystem should be much more predictable, with a lot more of the details hidden from the rest of the kernel. 

That is all work for a future series, though; the first step is to get the infrastructure set up. Chances are that will require at least one more iteration of Thomas's "Episode 1" patch set, meaning that it is unlikely to be 3.9 material. Starting around 3.10, though, we may well see significant changes to how CPU hotplugging is handled; the result should be more comprehensible and reliable code.
