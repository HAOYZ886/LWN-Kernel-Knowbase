---
title: "Rethinking power-aware scheduling"
url: https://lwn.net/Articles/474915/
date: "January 10, 2012"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet January 10, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 10, 2012

Sometimes it seems that there are few uncontroversial topics in kernel development, but saving power would normally be among them. Whether the concern is keeping a battery from running down too soon or keeping the planet from running down too soon, the ability to use less power per unit of computation is seen as a good thing. So when the kernel's scheduler maintainer threatened to rip out a bunch of power-saving code, it got some people's attention. 

The main thing the scheduler can do to reduce power consumption is to allow as many CPUs as possible to stay in a deep sleep state for as long as possible. With contemporary hardware, a comatose CPU draws almost no power at all. If there is a lot of CPU-intensive work to do, there will be obvious limits on how much sleeping the CPUs can get away with. But, if the system is lightly loaded, the way the scheduler distributes running processes can have a significant effect on both performance and power use. 

Since there is a bit of a performance tradeoff, the scheduler exports a couple of tuning knobs under `/sys/devices/system/cpu`. The first, called `sched_mc_power_savings`, has three possible settings: 

  0. The scheduler will not consider power usage when distributing tasks; instead, tasks will be distributed across the system for maximum performance. This is the default value. 

  1. One core will be filled with tasks before tasks will be moved to other cores. The idea is to concentrate the running tasks on a relatively small number of cores, allowing the others to remain idle. 

  2. Like (1), but with the additional tweak that newly awakened tasks will be directed toward "semi-idle" cores rather than started on an idle core. 

There is another knob, `sched_smt_power_savings`, that takes the same set of values, but applies the results to the threads of symmetric multithreading (SMT) processors instead. These threads look a lot like independent processors, but, since they share most of the underlying hardware, they are not truly independent from each other. 

Recently, Youquan Song noticed that `sched_smt_power_savings` did not actually work as advertised; a quick [patch](<https://lwn.net/Articles/474924/>) followed to fix the problem. Scheduler maintainer Peter Zijlstra objected to the fix, but he also made it clear that he objects to the power-saving machinery in general. Just to make that clear, he came back with [a patch](<https://lwn.net/Articles/474925/>) removing the whole thing and a threat to merge that patch unless somebody puts some effort into cleaning up the power-saving code. 

Peter subsequently made it clear that he sees the value of power-aware scheduling; the real problem is in the implementation. And, within that, the real problem seems to be the control knobs. The two knobs provide similar behavioral controls at two levels of the [scheduler domain](<https://lwn.net/Articles/80911/>) hierarchy. But, with three possible values for each, the result is nine different modes that the scheduler can run in. That seems like too much complexity for a situation where the real choice comes down to "run as fast as possible," or "use as little power as possible." 

In truth, it is not quite that simple. The performance cost of loading up every thread in an SMT processor is likely to be higher than that of concentrating tasks at higher levels. Those threads contend for the actual CPU hardware, so they will slow each other down. So one could conceive of situations where an administrator might want to enable different behavior at different levels, but such situations are likely to be quite rare. It is probably not worth the trouble of maintaining the infrastructure to support nine separate scheduler modes just in case somebody wants to do something special. 

For added fun, early versions of [the patch adding the "book" scheduling level](<http://permalink.gmane.org/gmane.linux.kernel/1022477>) (used only by the s390 architecture) included a `sched_book_power_savings` switch, though that switch went away before the patch was merged. There is also the looming possibility that somebody may want to do the same for scheduling at the NUMA node level. There comes a point where the number of possibilities becomes ridiculous. Some people - Peter, for example - think that point has already been reached. 

That conclusion leads naturally to talk of what should replace the current mechanism. One solution would be a simple knob with two settings: "performance" or "low power." It could, as Ingo Molnar [suggested](<https://lwn.net/Articles/474931/>), default to performance for line-connected systems and low power for systems on battery. That seems like a straightforward solution, but there is also [a completely different approach](<https://lwn.net/Articles/474932/>) suggested by Indan Zupancic: move that decision making into the CPU governor instead. The governor is charged with deciding which power state a CPU should be in at any given (idle) time. It could be given the additional task of deciding when CPUs should be taken offline entirely; the scheduler could then just do its normal job of distributing tasks among the CPUs that are available to it. Moving this responsibility to the governor is an interesting thought, but one which does not currently have any code to back it up; until somebody rectifies that little problem, a governor-based approach probably will not receive a whole lot more consideration. 

Somebody probably will come through with the single-knob approach, though; whether they will follow through and clean up the power-saving implementation within the scheduler is harder to say. But it should be enough to avert the threat of seeing that code removed altogether. And that is certainly a good thing; imagine the power that would be uselessly consumed in a flamewar over a regression in the kernel's power-aware scheduling ability.
