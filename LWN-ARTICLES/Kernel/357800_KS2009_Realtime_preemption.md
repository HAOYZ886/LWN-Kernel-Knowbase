---
title: "KS2009: Realtime preemption"
url: https://lwn.net/Articles/357800/
date: "October 21, 2009"
category: Realtime
author: "By Jonathan Corbet October 21, 2009 LWN's 2009 Kernel Summit coverage"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 21, 2009

* * *

LWN's [2009 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2009/>)

Thomas Gleixner led a session on the realtime preemption tree. The primary purpose of this discussion was to reach a conclusion on the naming of locking primitives in the kernel. Merging the realtime tree requires splitting the current `spinlock_t` type in two - a few "core spinlocks" which retain spinlock behavior even in realtime kernels, and the "sleeping spinlocks" which behave very differently in those kernels. As was discussed at the [realtime preemption mini-summit](<http://lwn.net/Articles/354690/>), the original plan was to rename just the core locks, for the simple reason that it's a much smaller patch. That, however, leaves the kernel with a "spinlock" type which might not be a spinlock; that naming created confusion at the mini-summit and was seen as being confusing for the rest of the development community as well. So the idea of changing `spinlock_t` to something like `lock_t` was taken to the kernel summit. 

The question got a quick answer: Linus has generally refrained from making outright decrees at this summit, but he did so here. He's very much against changing the name of spinlocks; it would be a huge patch, causing a lot of readjustment and pain, and he doesn't think it's worth it. Besides, Linus asserted, these locks remain spinlocks in all the ways that matter: code holding them still cannot schedule. Other parts of the kernel didn't change names when the realtime tree changed their function; code which looks like it is disabling interrupts may not be, for example. The same thing can be done here. 

And, according to Linus, the realtime people are crazy, so they can be left to deal with the weird stuff. 

There was some discussion, but few people were willing to argue against Linus on this one. So the decision appears to be made: `spinlock_t` will not change. Instead, the developers looked like they were heading into a full-on debate on the proper name for "core spinlocks" before being redirected to something more useful. 

Removal of the big kernel lock (BKL) was discussed for a bit. Perhaps the biggest remaining problem is in the TTY layer, where few developers dare to tread. The BKL is still needed there, but nobody is quite sure what it is protecting. Some (more) effort will be needed to clean that code up. Beyond that, much of the remaining BKL-removal work involves going into ancient drivers, deciding what needs to be done about locking, and clean them up. But there is a real suspicion that many of those drivers are not really in active use anymore, so it's not clear that it is worth the effort to fix them. There was talk of introducing a `CONFIG_BKL` configuration option which would be required to enable those drivers; that option could then be made to conflict with realtime operation. 

In general, the realtime preemption code is getting close to being ready to merge into the mainline. There are a few remaining issues, though. One of those is per-CPU data. The kernel makes extensive use of per-CPU variables; they can be accessed quickly, without locking, and without cache contention. Use of per-CPU data requires disabling preemption, though, which is exactly what the realtime developers are trying to get away from. The current realtime tree deals with the problem by protecting per-CPU data with (sleeping) locks, but that solution is seen as too ugly to push into the mainline. So something else will have to be done there, but what that "something else" will be is not yet clear. 

Significantly, nobody questioned the overall value of merging the realtime code into the mainline. Instead, some of the other discussions have made it clear that there are a lot of users for this functionality and that it is needed. So this merger will eventually happen, but your editor has learned better than to try to predict when. 

[Next: Generic architecture support](<https://lwn.net/Articles/357803/>)
