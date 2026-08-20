---
title: Ticket spinlocks
url: https://lwn.net/Articles/267968/
date: "February 6, 2008"
category: Spinlocks
author: "By Jonathan Corbet February 6, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 6, 2008

Spinlocks are the lowest-level mutual exclusion mechanism in the Linux kernel. As such, they have a great deal of influence over the safety and performance of the kernel, so it is not surprising that a great deal of optimization effort has gone into the various (architecture-specific) spinlock implementations. That does not mean that all of the work has been done, though; a patch merged for 2.6.25 shows that there is always more which can be done. 

On the x86 architecture, in the 2.6.24 kernel, a spinlock is represented by an integer value. A value of one indicates that the lock is available. The `spin_lock()` code works by decrementing the value (in a system-wide atomic manner), then looking to see whether the result is zero; if so, the lock has been successfully obtained. Should, instead, the result of the decrement option be negative, the `spin_lock()` code knows that the lock is owned by somebody else. So it busy-waits ("spins") in a tight loop until the value of the lock becomes positive; then it goes back to the beginning and tries again. 

Once the critical section has been executed, the owner of the lock releases it by setting it to 1\. 

This implementation is very fast, especially in the uncontended case (which is how things should be most of the time). It also makes it easy to see how bad the contention for a lock is - the more negative the value of the lock gets, the more processors are trying to acquire it. But there is one shortcoming with this approach: it is unfair. Once the lock is released, the first processor which is able to decrement it will be the new owner. There is no way to ensure that the processor which has been waiting the longest gets the lock first; in fact, the processor which just released the lock may, by virtue of owning that cache line, have an advantage should it decide to reacquire the lock quickly. 

One would hope that spinlock unfairness would not be a problem; usually, if there is serious contention for locks, that contention is a performance issue even before fairness is taken into account. Nick Piggin recently revisited this issue, though, after noticing: 

On an 8 core (2 socket) Opteron, spinlock unfairness is extremely noticable, with a userspace test having a difference of up to 2x runtime per thread, and some threads are starved or "unfairly" granted the lock up to 1 000 000 (!) times. 

This sort of runtime difference is certainly undesirable. But lock unfairness can also create latency issues; it is hard to give latency guarantees when the wait time for a spinlock can be arbitrarily long. 

Nick's [response](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=314cdbefd1fd0a7acf3780e9628465b77ea6a836>) was a new spinlock implementation which he calls "ticket spinlocks." Under the initial version of this patch, a spinlock became a 16-bit quantity, split into two bytes: 

> ![\[ticket spinlock\]](https://static.lwn.net/images/ns/kernel/ticket-spinlock.png)

Each byte can be thought of as a ticket number. If you have ever been to a store where customers take paper tickets to ensure that they are served in the order of arrival, you can think of the "next" field as being the number on the next ticket in the dispenser, while "owner" is the number appearing in the "now serving" display over the counter. 

So, in the new scheme, the value of a lock is initialized (both fields) to zero. `spin_lock()` starts by noting the value of the lock, then incrementing the "next" field - all in a single, atomic operation. If the value of "next" (before the increment) is equal to "owner," the lock has been obtained and work can continue. Otherwise the processor will spin, waiting until "owner" is incremented to the right value. In this scheme, releasing a lock is a simple matter of incrementing "owner." 

The implementation described above does have one small disadvantage in that it limits the number of processors to 256 - any more than that, and a heavily-contended lock could lead to multiple processors thinking they had the same ticket number. Needless to say, the resulting potential for mayhem is not something which can be tolerated. But the 256-processor limit is an unwelcome constraint for those working on large systems, which already have rather more processors than that. So the add-on ["big ticket" patch](<http://git.kernel.org/git/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=3a556b26a2718e48aa2b6ce06ea4875ddcd0778e>) \- also merged for 2.6.25 - uses 16-bit values when the configured maximum number of processors exceeds 256. That raises the maximum system size to 65536 processors - who could ever want more than that? 

With the older spinlock implementation, all processors contending for a lock fought to see who could grab it first. Now they wait nicely in line and grab the lock in the order of arrival. Multi-thread run times even out, and maximum latencies are reduced (and, more to the point, made deterministic). There is a slight cost to the new implementation, says Nick, but that gets very small on contemporary processors and is essentially zero relative to the cost of a cache miss - which is a common event when dealing with contended locks. The x86 maintainers clearly thought that the benefits of eliminating the unseemly scramble for spinlocks exceeded this small cost; it seems unlikely that others will disagree.
