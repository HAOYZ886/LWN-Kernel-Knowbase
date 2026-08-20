---
title: Greedy hrtimer expiration
url: https://lwn.net/Articles/461592/
date: "October 4, 2011"
category: Timers; hrtimer
author: "By Jonathan Corbet October 4, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 4, 2011

High-resolution timers (hrtimers) can be used to invoke kernel code after a precisely-specified time interval; unlike regular kernel timers, hrtimers can be reliably used with periods of microseconds or nanoseconds. Even hrtimer users can usually accept a wakeup within a specific range of times, though. To take advantage of that fact, the kernel offers "range hrtimers" with both soft (earliest) and hard (latest) deadlines. With range hrtimers, the kernel can coalesce wakeup events, minimizing the number of interrupts and reducing power usage. These are good things, so it is not surprising that the use of range timers has increased since they were introduced. 

One would think that, once the hrtimer code starts running in response to a timer interrupt, it would make sense to run every timer event whose soft expiration time has passed. But that is not what current kernels do. It is an interesting exercise to look at why that is, and how [a recent patch](<https://lwn.net/Articles/460523/>) from Venkatesh Pallipadi changes that behavior. 

For the sake of simplicity, let us imagine a set of timers that we'll call "A" through "G", each expiring 10µs after its predecessor. The hard expiration times are regular, but the timers have wildly differing soft expiration times; plotted on a timeline, the example timers look like this: 

> ![\[Hrtimer
timeline\]](https://static.lwn.net/images/2011/hrtimer-tl.png)

As can be seen here, timer "A" has a hard expiration 10µs in the future, but it could expire any time after 5µs. Timer "B" can be expired anytime from 7.5µs to 20µs in the future; the kernel can thus expire them both at 10µs and eliminate the need to schedule a timer interrupt at 20µs. Further in the future, timer "D" has a hard expiration 40µs ahead, but it is quite flexible and could, like timer "B", legitimately be expired 7.5µs from now. 

If the kernel is interrupted by a hardware timer in 10µs, it might be expected to call the expiration function for timers "A", "B", and "D". In reality, though, the expiration function for "D" will not be called at that time. To understand why, consider that hrtimers, within the kernel, are stored in a [red-black tree](<https://lwn.net/Articles/184495/>) with the hard expiration time as the key. The resulting tree will look something like this: 

> ![\[hrtimer
tree\]](https://static.lwn.net/images/2011/hrtimer-tree.png)

When the timer interrupt happens, the timer code performs a depth-first traversal of this tree for as long as it finds timers whose soft expiration time has passed. In this case, it will encounter "A" and "B" but, once it hits "C", the soft expiration time is in the future and the traversal stops. The organization of the data structure is such that the code cannot find the other events whose soft expiration time has passed without searching the whole tree. 

When the hrtimer code was extended to support range timers, searching for all soft-expired timers looked like it would require the addition of a second tree over the existing tree. That was deemed to be too expensive, especially since it may not actually save any wakeups. With the current code, "D" will be expired after 30µs, when "C" hits its hard expiration. Expiring "D" sooner will not eliminate the need for a wakeup at 30µs, so it didn't seem worth the effort to expire "D" sooner. 

Venkatesh thought this through and decided that he could come up with a couple of exceptions to that reasoning. It may well be that, at 10µs, the system will be less heavily loaded than at 30µs; in that case, it makes sense to get more work done sooner. Running the timer sooner also _could_ save a wakeup if "C" is deleted prior to expiration. So he wrote up a patch to implement a "greedy hrtimer walk" that would run all soft-expired hrtimers on a timer interrupt. 

He was helped by the addition of [augmented red-black trees](<https://lwn.net/Articles/388118/>) (also done by Venkatesh) in 2010. Essentially, an augmented tree allows the addition of a bit of extra metadata to each node; when a change is made to the tree, that extra information can be percolated upward. The greedy hrtimer walk patch turns the hrtimer tree into an augmented red-black tree; each node then stores the earliest soft expiration time to be found at that level of the tree or below. With the timer example given above, the new tree would look like this: 

> ![\[hrtimer
tree\]](https://static.lwn.net/images/2011/hrtimer-tree-aug.png)

The new numbers in red tell the tree-traversal logic what the soonest soft-expiration time is in each subtree. Using those numbers, a search of the tree 10µs in the future could prune the search at "F", since all soft expiration times will be known to be at least 25µs further in the future at that time. That takes away much of the cost of searching the tree for soft-expired timers that are not on the left side. 

One might still wonder if that extra work is worthwhile on the off-chance that running timer events sooner will be advantageous. After all, in the absence of specific knowledge or a crystal ball, it is just as likely that the system will be _less_ loaded at the later expiration time; in that case, expiring the timer sooner would make things worse. Venkatesh's patch avoids that issue by only performing the greedy hrtimer walk if the CPU is idle when the timer interrupt happens. If work is being done, soft-expired timers that are not immediately accessible are left in the tree, but, if the CPU has nothing better to do, it performs the full search. 

Venkatesh benchmarked this work by looking at the number of times the scheduler migrated tasks between CPUs on a given workload. Migrations are a sign of contention for the processor; they can also be expensive since processes can leave their memory cache behind when they move. Given the right workload (80% busy with a number of threads), the number of migrations was cut to just over half its previous value; other workloads gave less impressive results, but the patch never seemed to hurt. Given that, the comments on the patch were mostly focused on the details - like whether the greedy behavior should be controlled by a sysctl knob or not. Chances are this feature will show up in the 3.2 kernel.
