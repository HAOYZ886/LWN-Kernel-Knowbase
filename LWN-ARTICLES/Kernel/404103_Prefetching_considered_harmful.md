---
title: Prefetching considered harmful
url: https://lwn.net/Articles/404103/
date: "September 8, 2010"
category: Prefetch
author: "By Jonathan Corbet September 8, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 8, 2010

As anybody who has read [What every programmer should know about memory](<http://lwn.net/Articles/250967/>) knows, performance on contemporary systems is often dominated by cache behavior. A single cache miss can cause a processor stall lasting for hundreds of cycles. The kernel employs many tricks and techniques to optimize cache behavior, but, as is often the case with low-level optimization, it turns out that some of those tricks are not as helpful as had been thought. 

The kernel's linked list macros include a set of operators for iterating through a list. At the top of a list-processing loop, the macros will issue a prefetch operation for the next entry in the list. The hope is that, by the time one entry has been processed, the CPU will have fetched the following entry into its cache, avoiding a stall at the beginning of the next trip through the loop. It seems like the sort of micro-optimization which can only help, and nobody has looked closely at these prefetch operations for a long time - until now. Andi Kleen has just posted [a patch](<http://lwn.net/Articles/404033/>) removing most of those prefetches. 

Andi's contention is that, on contemporary processors, the prefetch operations are actually making things worse. These processors already prefetch everything they can get their hands on, so the explicit prefetch is unlikely to help. Even if that prefetch does start a memory cycle earlier than it would have otherwise happened, list processing loops tend to be so short that the amount of additional parallelism gained is quite small. Meanwhile the prefetch operations bloat the kernel image, increase register use, and cause the compiler to generate worse code. So, he says, we are better off without them. 

With the prefetch operations removed, Andi's kernel image ends up being 10KB smaller. It also shows no performance regressions over mainline kernels. Unless somebody else gets different results, that seems like enough to justify putting this patch into the mainline.
