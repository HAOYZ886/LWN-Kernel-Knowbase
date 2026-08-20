---
title: The balance between features and performance in the block layer
url: https://lwn.net/Articles/874643/
date: "November 5, 2021"
category: "Block layer-Scalability; Scalability"
author: "By Jonathan Corbet November 5, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 5, 2021

Back in September, LWN [reported](<https://lwn.net/Articles/868070/>) on a series of block-layer optimizations that enabled a suitably equipped system to sustain 3.5 million I/O operations per second (IOPS). That optimization work has continued since then, and those 3.5 million IOPS would be a deeply disappointing result now. A recent disagreement over the addition of a new feature has highlighted the potential cost of a heavily optimized block layer, though; when is a feature deemed important enough to outweigh the drive for maximum performance? 

Block subsystem maintainer Jens Axboe has continued working to make block I/O operations go faster. [A recent round of patches](<https://lwn.net/ml/linux-block/20211017013748.76461-1-axboe@kernel.dk/>) tweaked various fast paths, changed the plugging mechanism to use a singly linked list, and made various other little changes. Each is a small optimization, but the work adds up over time; the claimed level of performance is now 8.2 million IOPS — well over September's rate, which looked good at the time. This work has since found its way into the mainline as part of the [block pull request](<https://lwn.net/ml/linux-block/f32307c6-5b97-00f6-3738-0732d3de5e62@kernel.dk/>) for 5.16. 

So far, so good; few people will argue with massive performance improvements. But they might argue with changes that threaten to interfere, even in a tiny way, with those improvements. 

Consider, for example, [this patch set](<https://lwn.net/ml/linux-kernel/20211021001059.438843-1-jane.chu@oracle.com/>) from Jane Chu. It adds a new flag (`RWF_RECOVERY_DATA`) to the [`preadv2()` and `pwritev2()` system calls](<https://man7.org/linux/man-pages/man2/writev.2.html>) that can be used by applications trying to recover from nonvolatile-memory "poisoning". Implementations of nonvolatile memory have different ways of detecting and coping with data corruption; Intel memory, it seems, will poison the affected range, meaning that it cannot be accessed without generating an error (which then turns into a `SIGBUS` signal). An application can respond to that error by reading or writing the poisoned range with the new flag; a read will replace the poisoned data with zeroes (allowing the application to obtain whatever data is still readable), while a write will overwrite that data and attempt to clear the poisoned status. Either way, the application can attempt to recover from the problem and continue running. 

Christoph Hellwig [objected](<https://lwn.net/ml/linux-kernel/YXj2lwrxRxHdr4hb@infradead.org/>) to this new flag on the grounds that it would slow down the I/O fast path: 

> Well, my point is doing recovery from bit errors is by definition not the fast path. Which is why I'd rather keep it away from the pmem read/write fast path, which also happens to be the (much more important) non-pmem read/write path. 

Pavel Begunkov also [complained](<https://lwn.net/ml/linux-kernel/22255117-52de-4b2d-822e-b4bc50bbc52b@gmail.com/>), saying that each flag adds a bit of overhead that piles up over time: ""default config kernels are already sluggish when it comes to really fast devices and it's not getting better"". That caused Darrick Wong to [ask](<https://lwn.net/ml/linux-kernel/20211029165747.GC2237511@magnolia/>): ""So we can't have data recovery because moving fast [is] the only goal?"". Begunkov [denied](<https://lwn.net/ml/linux-kernel/f3e14569-a399-f6da-fd3e-993b579eaf74@gmail.com/>) saying that, but wasn't really clear on what he _was_ saying. 

The cost of this flag is tiny — perhaps not even measurable — in cases where it is not used. But even that overhead can look unacceptable to developers who are working to get the sustained IOPS numbers as high as possible. One flag leads to another and another, and someday in the future the performance cost becomes significant — or that is the argument, anyway. To avoid this kind of problem, the argument continues, niche features like nonvolatile memory poison recovery should be restricted to parts of the kernel that are not seen as being the fast path. In this case, adding the needed functionality to `fallocate()` has been suggested [and tried](<https://lwn.net/ml/linux-xfs/163192864476.417973.143014658064006895.stgit@magnolia/>), but it was eventually decided that `fallocate()` is not the right place for hardware-management features like poison-clearing. 

Thus the current implementation, which has run into fast-path concerns. That, in turn, has provoked [an extended outburst](<https://lwn.net/ml/linux-kernel/20211029223233.GB449541@dread.disaster.area/>) from Dave Chinner, who thinks that much of the current optimization work is misplaced: 

> The current approach of hyper-optimising the IO path for maximum per-core IOPS at the expense of everything else has been proven in the past to be exactly the wrong approach to be taking for IO path development. Yes, we need to be concerned about performance and work to improve it, but we should not be doing that at the cost of everything else that the IO stack needs to be able to do. 

Optimization, he continued, should come after the needed functionality is present; ""using 'fast path optimisation' as a blunt force implement to prevent new features from being implemented is just ... obnoxious"". 

The conversation stalled out shortly after that. This kind of disagreement over features and performance has been heard in the kernel community many times over the decades, though. Going faster is a popular goal, and the developers who are focused on performance have been known to get tired of working through performance regressions caused by feature additions that accumulate over time. But functionality, too, is important, and developers tire of having needed features blocked on seemingly insignificant performance concerns. 

Often, performance-related pushback leads to revised solutions that do not have the same performance cost; along those lines, it's worth noting that Hellwig [has some ideas](<https://lwn.net/ml/linux-kernel/YYDYUCCiEPXhZEw0@infradead.org/>) for improved ways of handling I/O to nonvolatile memory. Other times, it just leads to the delay or outright blocking of needed functionality. What will happen in this case is not clear at this point, but debates like this will almost certainly be with us in the coming decades as well. It is, in the end, how the right balance between performance and functionality is (hopefully) found.
