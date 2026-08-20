---
title: Realtime mainlining
url: https://lwn.net/Articles/662833/
date: "November 3, 2015"
category: Realtime
author: "By Jonathan Corbet November 3, 2015 2015 Kernel Summit"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 3, 2015

* * *

[2015 Kernel Summit](<https://lwn.net/Articles/KernelSummit2015/>)

The Linux Foundation [announced](<https://lwn.net/Articles/659193/>) in early October that it had pulled together a collaborative project to support work on the realtime kernel patches. Thomas Gleixner, who has been appointed as a Linux Foundation fellow to lead that work, ran a session at the 2015 Kernel Summit to discuss his plans for realtime Linux in the coming year or so. 

The first item on the list would appear to be restarting the work to fix the CPU hotplugging code. Adding or removing a CPU is currently handled by "a notifier mess" that is hard to follow and which implements the plug and unplug operations in an asymmetric manner ("asymmetric" meaning that the steps in the plug and unplug procedures do not match up with each other). It is, Thomas said, the biggest mess that is getting in the way of realtime work at the moment. He had posted [a plan for a new CPU hotplug subsystem](<https://lwn.net/Articles/537562/>) in 2013; that plan will be picked back up and pushed forward. 

The new mechanism involves a (symmetric) array of states in place of the notifier-based scheme. Thomas also plans to move as much code as possible out of the architecture-specific trees and into the core kernel; there are, he said, a lot of duplicated and needlessly inconsistent implementations out there now. Most of the callbacks for a new CPU will be moved to the CPU that is coming up; at the moment, they run on the CPU that initiates the hotplug operation while the new CPU sits and waits. 

In summary, CPU hotplug is a nightmare for realtime and for the mainline kernel as well. It breaks regularly and gets fixed with another layer of duct tape; nobody can really say how it works. It is, he said, "25 years of duct tape" that needs to be bulldozed and reimplemented from the beginning. He will do exactly [![\[Thomas Gleixner\]](https://static.lwn.net/images/conf/2015/klf-ks/ThomasGleixner-sm.jpg)](<https://lwn.net/Articles/662838/>) that, and hopes not to break anything in the process. While he is at it, he should be able to make many of the state-transition operations able to run in parallel, speeding the whole thing. The new scheme will also make it possible to intensively test each transition between the (many) states, something that cannot be done now. It will, he said, be "a lot of patches." 

The next item on the agenda is to finish the [timer wheel reimplementation](<https://lwn.net/Articles/646950/>). He has collected a lot of data on how the timer wheel is actually used; timeouts tend to be spread out evenly up to about 1ms into the future. After that, they are widely scattered and irregular. 93% (or more) timeouts added to the wheel never expire. Given that, the expensive cascading work done by the timer wheel now is mostly wasted. 

Thomas's approach is to reduce the granularity of timeouts scheduled far into the future. What that means is that far-future timeouts may expire significantly later than intended; as a general rule, this imprecision is seen as being harmless. There was some concern about adding inaccuracy to user-space timers, but those are all implemented with high-resolution timers in the kernel and do not go through the timer wheel at all. Making far-future kernel timers less accurate seems to make a few kernel developers uncomfortable, but nobody pointed to anything that would actually break. 

Sometime next year Thomas plans to get around to what he called the "per-CPU big kernel locks" — the overuse of preemption disabling and local interrupt masking. Disabling preemption has obvious implications for realtime performance, since code that has disabled preemption cannot be preempted by a higher-priority process. But disabling preemption has another problem, in that it is often not clear what is being protected. There is no actual lock involved, so a disabling operation is not tied to any particular data structure. That makes the code hard to change with any sort of confidence. 

The plan is to add a new kind of lock, which goes by the name `local_lock` in the realtime tree — Thomas said he was open to discussions regarding better names. These locks can be tied to the data structures they protect; those are per-CPU variables most of the time. In the realtime tree, a `local_lock` is a real lock; in mainline, it will simply disable preemption as is done now. The end result will be a solution to the realtime issue, clearer locking that can be verified with the lockdep tool, and, perhaps, the eventual ability to check for missing locking with the `sparse` utility. 

As Thomas reached the end of his talk, Andy Lutomirski asked whether the [migration-disable approach](<https://lwn.net/Articles/452884/>) would move from the realtime tree to the mainline. The realtime tree is often able to disable migration in places where preemption is disabled in mainline, preserving the ability to preempt the running process. But, Thomas said, the immediate plan is not to move that code to mainline; there are other problems to solve first. Andy would like to see it there so that he can avoid creating future work by adding `preempt_disable()` calls now. Peter Zijlstra, though, said that disabling migration creates problems for the scheduler; he would rather just have processes disabling preemption. So the migration-disable patches seem likely to stay in the realtime tree for a while yet.
