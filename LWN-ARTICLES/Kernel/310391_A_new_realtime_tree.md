---
title: A new realtime tree
url: https://lwn.net/Articles/310391/
date: "December 9, 2008"
category: Realtime; Spinlocks
author: "By Jonathan Corbet December 9, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 9, 2008

It has been just over four years, now, since [the realtime discussion got serious](<http://lwn.net/Articles/106010/>) and the realtime preemption patch set got its start. During that time, your editor has heard many predictions for when the bulk of the realtime work would be merged; generally, the guess has been "within about a year." While a lot of realtime work _has_ been merged, some of the core components of the realtime tree remain outside of the mainline. Beyond that, the realtime developers have been relatively quiet over the last year - at least on the realtime front. Having taken on some little side tasks - unifying the x86 architecture and maintaining it going forward, for example \- some of those developers have been just a little bit distracted recently. 

The realtime patch set has not gone away, though. If nothing else, the fact that a number of distributors are shipping this code is enough to ensure continued interest in its development. So your editor noted with interest the recent [announcement of a new -rt tree](<http://lwn.net/Articles/310368/>) with an updated set of realtime patches. This tree will be of interest for anybody wanting to look at the realtime work in the context of the 2.6.28 kernel or beyond. 

One of the core technologies in the realtime tree is a change to how spinlocks work. Spinlocks in the mainline will busy-wait until the required lock becomes available; they thus occupy the processor to no useful end when acquiring a contended lock. Holding a spinlock will also prevent a thread from being preempted. This behavior is generally best for system throughput; it also makes it easier to write correct code. But anything which prevents a CPU from immediately servicing the highest-priority process runs counter to the chief design goal of a realtime operating system: providing deterministic response times in all situations. So, for the realtime patches, classic spinlocks had to go. 

The solution was to turn most spinlocks into a form of mutex with priority inheritance. A process which attempts to acquire a contended "spinlock" will no longer spin; instead, it goes to sleep and waits for the lock to become free, making the processor available to another thread. Code which holds one of these non-spinlocks is no longer immune to preemption; a higher-priority thread can always push it out of the way. By changing spinlocks in this way, the realtime hackers were able to eliminate one of the largest sources of latency in the mainline kernel. Much of that work found its way into the mainline some time ago in the form of the mutex API, but spinlocks themselves have not been changed in the mainline. 

To minimize the pain of maintaining the realtime patches, the developers simply redefined the `spinlock_t` type to be the new mutex type instead. Except that, as it turns out, some spinlocks in low-level parts of the kernel really do need to be spinlocks still. So those were switched to a new `raw_spinlock_t` type - but without changing the various `spin_lock()` calls. Instead, some truly frightening macro trickery was introduced to cause the spinlock API to do the right thing when passed either of two entirely different mutual exclusion primitives. This bit of macro magic was always going to be an impediment to mainline inclusion, so the realtime developers never really expected to merge the lock code in that form. 

The new realtime tree now shows how the realtime developers think this work might get into the mainline. It involves a more explicit separation of the two types of "spinlocks" - and a lot of code churn. In the realtime tree, most locks of type `spinlock_t` are changed to a new `lock_t` type. There is a new set of operations for this type: 

```
#include <linux/lock.h>
    
        lock_t lock;
    
        acquire_lock(&lock);
        release_lock(&lock);
```

For a normal, non-realtime kernel build, `lock_t` will be the same as `spinlock_t`, and things will work as they always have. On realtime kernels, instead, `lock_t` will be a mutex type. The other variants of the spinlock API will be represented in the new API (there is an `acquire_lock_irqsave()`, for example), but none of them will actually disable interrupts in a realtime kernel. Meanwhile, `spinlock_t` will remain a true spinlock type. 

This change gets rid of the tricky macros, but at the cost of changing the declarations of and operations on almost all spinlocks in the kernel. That is a lot of code changes: a quick grep turns up over 20,000 `spin_lock*()` calls in the upcoming 2.6.28 kernel. That will make for some pain if and when this change is merged. But in the mean time, it can only make for a _lot_ of pain for the people who have to maintain this patch out of tree. To make their lives a little easier, the realtime developers have created a couple of scripts to do the bulk of the work. First, all spinlocks in a pristine kernel are converted to `lock_t`, then the few locks which truly must be spinlocks are switched back. This work is kept in a separate branch which is regenerated when needed; in this way, the realtime developers avoid the need to do nasty merges to keep up with current kernels. 

Your editor has heard talk of another locking change which does not, yet, appear in this tree. One problem with the realtime patch set is that it requires distributors to create yet another kernel build - something they hate doing - if they want to support realtime operation. In an effort to make life easier for distributors, the realtime developers are working on a scheme whereby a kernel would determine at run time whether it should be running in a realtime mode. If so, spinlocks will be changed to sleeping locks by patching the kernel binary as it boots. Kernels built this way will be able to run efficiently in either mode. 

The branches of the realtime tree provide a quick guide to the other parts of the realtime work which remain outside of the mainline. The [threaded interrupt handler](<http://lwn.net/Articles/302043/>) code is one example; that change could be proposed (again) for merging in the near future. The [priority workqueue](<http://lwn.net/Articles/243920/>) mechanism sits in another branch, as do patches aimed at Java support, filesystem changes, memory management changes, and more. Then, there's a branch for stuff which will never be merged; for example, there is [this patch](<http://git.kernel.org/?p=linux/kernel/git/rostedt/linux-2.6-rt.git;a=commitdiff;h=54db6a0c8b2cc7b07ad46cb6b533484b913306f6;hp=721d5dfe7e516954c501d5e9d0dfab379cf4241a>) which gives Java programs direct access to physical memory - not something which strikes most kernel developers as a good idea. All told, there is a great deal of work sitting in the realtime patch set; this work is finally being organized into a proper git tree. 

The "upstream first" policy says that vendors should merge their code upstream before shipping it to customers. The 2.6.x development model is built on the idea that no change is too fundamental to be accepted into a regular, 3-month development cycle. The realtime patches would appear to be an exception to both rules. It has taken over four years to get to a point where some of the fundamental realtime technologies are close to ready for the mainline, but distributors have been shipping it for at least three of those years. It has, in other words, been one of the biggest forks of the Linux kernel, ever. The plan has always been to join this fork back with the mainline, though; perhaps, finally, that goal is getting closer. With luck, it will happen within about a year.
