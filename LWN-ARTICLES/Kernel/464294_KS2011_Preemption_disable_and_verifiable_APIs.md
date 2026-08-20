---
title: "KS2011: Preemption disable and verifiable APIs"
url: https://lwn.net/Articles/464294/
date: "October 24, 2011"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet October 24, 2011 2011 Kernel Summit coverage"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 24, 2011

* * *

[2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) coverage 

Thomas Gleixner is grumpy about some of the interfaces that have been added to the kernel recently, and he used a session at the 2011 Kernel Summit to talk about it. When we are not careful, he said, we add code to the kernel which destabilizes it and makes problems harder to find. 

The specific problem was the `this_cpu_*()` series of functions which are intended to make access to per-CPU data faster. These functions were also discussed at the just-concluded [Realtime Summit](<https://lwn.net/Articles/464180/>); see that report for a more detailed summary of the problems. In short: these functions work without disabling preemption, making the concept of "this CPU" a slippery one at best. Some of them, like `this_cpu_write()`, simply cannot be used in a correct manner. 

Thomas wants, at a minimum, to add some debugging infrastructure that would make it clear what data is being operated on and when. Beyond that, though, he said that a lot of calls to `preempt_disable()` are popping up throughout the kernel. Disabling preemption can make access to certain types of data safe, but it is, once again, not really clear what is being protected. According to Thomas, `preempt_disable()` is the new big kernel lock. Peter added that it is a sort of big per-CPU data lock, but there is no desire or need to take a big lock. It is far better to lock the specific data at hand so that multiple users of different data structures can be interleaved in the scheduler if need be. 

Linus responded that he has no problem with a per-CPU data lock that disappears in mainline kernels and allows verification of locking with lockdep. But, he said, calling it a new big kernel lock is a bit unfair; it isn't quite that bad. And, in any case, the big kernel lock worked for the kernel for many years. Andi Kleen worried that using lockdep with per-CPU data could lead to spurious warnings in places where the "lock" ordering does not really matter. 

Ted Ts'o asked about the impact of the bugs that had been found. Thomas responded that most of them were in statistical functions and, thus, did not matter much. But there was one in the filesystem layer in 2.6.38 or 2.6.39 that would store a pointer on the wrong CPU and hose a disk, and another one that made the SLUB allocator explode. If, he said, even the people who created this interface (the `this_cpu_*()` functions were initially added for use with SLUB) cannot get its use right, something is really wrong. That is scary since use of these functions is exploding throughout the kernel. 

Some changes are in the works. They may involve renaming the functions and will almost certainly involve the removal of the most dangerous ones and the addition of some debugging infrastructure. 

Next: [Scheduler testing](<https://lwn.net/Articles/464296/>)
