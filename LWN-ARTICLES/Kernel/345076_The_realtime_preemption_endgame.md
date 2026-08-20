---
title: The realtime preemption endgame
url: https://lwn.net/Articles/345076/
date: "August 5, 2009"
category: Atomic spinlocks; Realtime; Spinlocks
author: "By Jonathan Corbet August 5, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 5, 2009

There has been relatively little noise out of the realtime preemption camp in recent months. That does not mean that the realtime developers have been idle, though; instead, they are preparing for the realtime endgame: the merger of the bulk of the remaining patches into the mainline kernel. The [2.6.31-rc4-rt1 tree](<http://lwn.net/Articles/344439/>) recently announced by Thomas Gleixner shows the results of much of this work. This article will look at some of the recent changes to -rt. 

The point of the realtime preemption project is to enable a general-purpose Linux kernel to provide deterministic response times to high-priority processes. "Realtime" does not (necessarily) mean "fast"; it means knowing for sure that the system can respond to important events within a specific time period. It has often been said that this cannot be done, that the complexity of a full operating system would thwart any attempt to guarantee bounded response times. Of course, it was also said that free software developers could never create a full operating system in the first place. The realtime hackers believe that both claims are equally false, and they have been working to prove it. 

One of the long-term realtime features was threaded interrupt handlers. A "hard" interrupt handler can monopolize the CPU for as long as it runs; that can create latencies for other users. Moving interrupt handlers into their own threads, instead, allows them to be scheduled like any other process on the system. Thus, threaded interrupt handlers cannot get in the way of higher-priority processes. 

Much of the threaded interrupt handling code moved into the mainline for the 2.6.30 release, but in a somewhat different form. While the threading of interrupt handlers is nearly universal in a realtime kernel, it's an optional (and, thus far, little-used) feature in the mainline, so the APIs had to change somewhat. Realtime interrupt handling has been reworked on top of the mainline threaded interrupt mechanism, but it still has its own twists. 

In particular, the kernel can still be configured to force all interrupt handlers into threads. If a given driver explicitly requests a threaded handler, behavior is similar to a non-realtime kernel; the driver's "hard" interrupt handler runs as usual in IRQ context. Drivers which do not request threaded handlers get one anyway, with a special hard handler which masks the interrupt line while the driver's handler runs. Interrupt handler threads are per-device now (rather than per-IRQ line). All told, the amount of code which is specific to the realtime tree is fairly small now; the bulk of it is in the mainline. 

Software interrupt handling is somewhat different in the realtime tree. Mainline kernels will normally handle software interrupts at convenient moments - context switches or when returning to user space from a system call, usually. If the software interrupt load gets too heavy, though, handling will be deferred to the per-CPU "ksoftirqd" thread. In the realtime tree (subject to a configuration option), all software interrupt handling goes into ksoftirqd - but now there is a separate thread for each interrupt line. So each CPU will get a couple of ksoftirqd threads for network processing, one for the block subsystem, one for RCU, one for tasklets, and so on. Software interrupts are also preemptable, though that may not happen very often; they run at realtime priority. 

The work which first kicked off the realtime preemption tree was the replacement of spinlocks with sleeping mutexes. The spinlock technique is difficult to square with deterministic latencies; any processor which is spinning on a lock will wait an arbitrary period of time, depending on what code in another CPU is doing. Code holding spinlocks also cannot be preempted; doing so would cause serious latencies (at best) or deadlocks. So the goal of ensuring bounded response times required the elimination of spinlocks to the greatest extent possible. 

Replacing spinlocks throughout the kernel with realtime mutexes solves much of the problem. Threads waiting for a mutex will sleep, freeing the processor for some other task. Threads holding mutexes can be preempted if a higher-priority process comes along. So, if the priorities have been set properly, there should be little in the way of the highest-priority process being able to respond to events at any time. This is the core idea behind the entire realtime preemption concept. 

As it happens, though, not all spinlocks can be replaced by mutexes. At the lowest levels of the system, there is still a need for true (or "raw") spinlocks; the locks which are used to implement mutexes are one obvious example. Over the years, a fair amount of effort has gone into the task of figuring out which spinlocks really needed to be "raw" locks. At the code level, the difference was papered over through the use of some rather ugly trickery in the spinlock primitives. Regardless of whether a raw spinlock or a sleeping lock was being used, the code would call `spin_lock()` to acquire it; the only difference was where the lock was declared. 

This approach was probably useful during the early development phases where it was often necessary to change the type of specific locks. But ugly compiler trickery which serves to obfuscate the type of lock being used in any specific context seems unlikely to fly when it comes to merger into the mainline. So the realtime hackers have bitten the bullet and split the two types of locks entirely. The replacement of "spinlocks" with mutexes still happens as before, for the simple reason that changing every spinlock call would be a massive, disruptive change across the entire kernel code base. But the "raw" spinlock type, which is used in far fewer places, is more amenable to this kind of change. 

The result is a new mutual exclusion primitive, called `atomic_spinlock_t`, which looks a lot like traditional spinlocks: 

```
#include <linux/spinlock.h>
    
        DEFINE_ATOMIC_SPINLOCK(name)
        atomic_spin_lock_init(atomic_spinlock_t *lock);
    
        void atomic_spin_lock(atomic_spinlock_t *lock);    
        void atomic_spin_lock_irqsave(atomic_spinlock_t *lock, long flags);
        void atomic_spin_lock_irq(atomic_spinlock_t *lock);
        void atomic_spin_lock_bh(atomic_spinlock_t *lock);
        int atomic_spin_trylock(atomic_spinlock_t *lock);    
    
        void atomic_spin_unlock(atomic_spinlock_t *lock);
        void atomic_spin_unlock_irqrestore(atomic_spinlock_t *lock, long flags);
        void atomic_spin_unlock_irq(atomic_spinlock_t *lock);
        void atomic_spin_unlock_bh(atomic_spinlock_t *lock);
```

These new "atomic spinlocks" are used in the scheduler, low-level interrupt handling code, clock-handling, PCI bus management, ACPI subsystem, and in many other places. The change is still large and disruptive - but much less so than changing ordinary "spinlock" users would have been. 

[PULL QUOTE:  One might argue that putting atomic spinlocks back into the kernel will reintroduce the same latency problems that the realtime developers are working to get rid of.  END QUOTE] One might argue that putting atomic spinlocks back into the kernel will reintroduce the same latency problems that the realtime developers are working to get rid of. There is certainly a risk of that happening, but it can be minimized with due care. Auditing every kernel path which uses spinlocks is clearly not a feasible task, but it _is_ possible to look very closely at the (much smaller) number of code paths using atomic spinlocks. So there can be a reasonable degree of assurance that the remaining atomic spinlocks will not cause the kernel to exceed the latency goals. 

(As an aside, Thomas Gleixner is [looking for a better name](<http://rt.wiki.kernel.org/index.php/Atomic_Spinlock>) for the `atomic_spinlock_t` type. Suggest the winning idea, and free beer at the next conference may be your reward.) 

Similar changes have been made to a number of other kernel mutual exclusion mechanisms. There is a new `atomic_seqlock_t` variant on [seqlocks](<http://lwn.net/Articles/22818/>) for cases where the seqlock writer cannot be preemptable. The `anon_semaphore` type mostly appears to be a renaming of semaphores and their related functions; it is a part of the continuing effort to eliminate the use of semaphores in any place where a mutex or completion should be used instead. There is also a `rw_anon_semaphore` type as a replacement for `rw_semaphore`. 

Quite a few other realtime-specific changes remain in the -rt tree. The realtime code is incompatible with the SLUB allocator, so only slab is allowed. There is also an interesting problem with `kmap_atomic()`; this function creates a temporary, per-CPU kernel-space address mapping for a given memory page. Preemption cannot be allowed to happen when an atomic kmap is active; it would be possible for other code to change the mapping before the preempted code tries to use it. In the realtime setting, the performance benefits from atomic kmaps are outweighed by the additional latency they can cause. So, for all practical purposes, `kmap_atomic()` does not exist in a realtime kernel; calls to `kmap_atomic()` are mapped to ordinary `kmap()` calls. And so on. 

As for work which is not yet even in the realtime tree, the first priority would appear to be clear: 

We seriously want to tackle the elimination of the PREEMPT_RT annoyance #1, aka BKL. The Big Kernel Lock is still used in ~330 files all across the kernel. 

At this point, the remaining BKL-removal work comes down to low-level audits of individual filesystems and drivers; for the most part, it has been pushed out of the core kernel. 

Beyond that, of course, there is the little task of getting as much of this code as possible into the mainline kernel. To that end, a proper git tree with a bisectable sequence of patches is being prepared, though that work is not yet complete. There will also be a gathering of realtime Linux developers at the [Eleventh Real-Time Linux Workshop](<http://www.osadl.org/RTLWS-2009.rtlws-2009.0.html>) this September in Dresden; getting the realtime work into the mainline is expected to be discussed seriously there. As it happens, your editor plans to be in the room; watch this space in late September for an update.
