---
title: Eliminating rwlocks and IRQF_DISABLED
url: https://lwn.net/Articles/364583/
date: "December 1, 2009"
category: "Interrupts; Reader-writer spinlocks; Spinlocks"
author: "By Jonathan Corbet December 1, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
December 1, 2009

Reader-writer spinlocks and interrupt-enabled interrupt handlers both have a long history in the Linux kernel. But both may be nearing the end of their story. This article looks at the push for the removal of a pair of legacy techniques for mutual exclusion in the kernel. 

Reader-writer spinlocks (rwlocks) behave like ordinary spinlocks, but with some significant exceptions. Any number of readers can hold the lock at any given time; this allows multiple processors to access a shared data structure if none of them are making changes to it. Reader locks are also naturally nestable; a single processor can acquire a given read lock more than once if need be. Writers, instead, require exclusive access; before a write lock can be granted, all read locks must be released, and only one write lock can be held at any given time. 

Rwlocks in Linux are inherently unfair in that readers can stall writers for an arbitrary period of time. New read locks are allowed even if a writer is waiting, so a steady stream of readers can block a writer indefinitely. In practice this problem rarely surfaces, but Nick Piggin has [reported](<https://lwn.net/Articles/364585/>) a case where the right user-space workload can cause an indefinite system livelock. This is a performance problem for specific users, but it is also a potential denial of service attack vector on many systems. In response, Nick started pondering on the challenge of implementing more fair rwlocks which do not create performance regressions. 

That is not an easy task. The obvious solution - blocking new readers when a writer gets in line - will not work for the most important rwlock (`tasklist_lock`) because that lock can be acquired by interrupt handlers. If a processor already holding a read lock on `tasklist_lock` is interrupted, and the interrupt handler, too, needs that lock, forcing the handler to wait will deadlock the processor. So workable solutions require allowing nested reader locks to be acquired even when writers are waiting, or disabling interrupts when `tasklist_lock` is held. Neither solution is entirely pleasing. 

Beyond that, there has been a general sentiment toward the removal of rwlocks for some years. The locking primitives themselves are significantly slower than plain spinlocks, so any performance gain from allowing multiple readers must be large enough to make up for that extra cost. In many cases, that gain does not appear to actually exist. So, over time, kernel developers have been changing rwlocks to normal spinlocks or replacing them with read-copy-update mechanisms. Even so, a few hundred rwlocks remain in the kernel. Perhaps it would be better to focus on removing them instead of putting a lot of work into making them more fair. 

Almost all of those rwlocks could be turned into spinlocks tomorrow and nobody would ever notice. But `tasklist_lock` is a bit of a thorny problem; it is acquired in many places in the core kernel and it's not always clear what this lock is protecting. This lock is also taken in a number of critical kernel fast paths, so any change has to be done carefully to avoid performance regressions. For these reasons, kernel developers have generally avoided messing with `tasklist_lock`. 

Even so, it would appear that, over time, a number of the structures protected by `tasklist_lock` have been shifted to other protection mechanisms. This lock has also been changed in the realtime preemption tree, though that code has not yet made it to the mainline. Seeing all this, Thomas Gleixner [decided to try to get rid of this lock](<https://lwn.net/Articles/364601/>), saying ""If nobody beats me I'm going to let sed loose on the kernel, lift the task_struct rcu free code from -rt and figure out what explodes."" As of this writing, the results of this exercise have not been posted. But Thomas is still active on the mailing list, so one concludes that any explosions experienced cannot have been fatal. 

If `tasklist_lock` can be converted successfully to an ordinary spinlock, the conversion of the remaining rwlocks is likely to happen quickly. Shortly after that, rwlocks may go away altogether, simplifying the set of mutual exclusion primitives in Linux considerably. 

#### IRQF_DISABLED

Meanwhile, a different sort of exclusion happens with interrupt handlers. In the early days of Linux, these handlers were divided into "fast" and "slow" varieties. Fast handlers could be run with other interrupts disabled, but slow handlers needed to have other interrupts enabled. Otherwise, a slow handler (perhaps doing a significant amount of work in the handler itself) could block the processing of more important interrupts, impacting the performance of the system. 

Over the years, this distinction has slowly faded away, for a number of reasons. The increase in processor speeds means that even an interrupt handler which does a fair amount of work can be "fast." Hardware has gotten smarter, minimizing the amount of work which absolutely must be done immediately on receipt of the interrupt. The kernel has gained improved mechanisms (threaded interrupt handlers, tasklets, and workqueues) for deferred processing. And the quality of drivers has generally improved. So driver authors generally do not really even need to think about whether their handlers run with interrupts enabled or not. 

Those authors still need to make that choice when setting up interrupt handlers, though. Unless the handler is established with the `IRQF_DISABLED` flag set, it will be run with interrupts enabled. For added fun, handlers for shared interrupts (perhaps the majority on most systems) can never be assured of running with interrupts disabled; other handlers running on the same interrupt line might enable them at any time. So many handlers will be running with interrupts enabled, even though that is not needed. 

The solution, it would seem, would be to eliminate the `IRQF_DISABLED` flag and just run all handlers with interrupts disabled. In almost all cases, everything will work just fine. There are just a few situations where interrupt handling still takes too long, or where one interrupt handler depends on interrupts for another device being delivered at any time. Those handlers could be identified and dealt with. "Dealt with" in this case could take a few forms. One would be to equip the driver with a better-written interrupt handler which does not have this problem. Another, related approach would be to move the driver to a threaded handler which, naturally, will run with interrupts enabled. Or, finally, the handler could be set up with a new flag (`IRQF_NEEDS_IRQS_ENABLED`, perhaps) which would cause it to run with interrupts turned on in the old way. 

It's not clear when all this might happen, but it could be that, in the near future, all hard interrupt handlers are expected to run - quickly - with interrupts disabled. Few people will even notice, aside from some maintainers of out-of-tree drivers who will need to remove `IRQF_DISABLED` from their code. But the kernel as a whole should be faster for it.
