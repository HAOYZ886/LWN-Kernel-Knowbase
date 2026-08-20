---
title: Writeout throttling
url: https://lwn.net/Articles/261626/
date: "December 11, 2007"
category: "Memory management-Writeout throttling; Write throttling"
author: "By Jonathan Corbet December 11, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 11, 2007

The avoidance of writeout deadlocks is a topic which occasionally pops up on the mailing lists. Most Linux systems are able to handle the writeout of dirty pages to disk without a great deal of trouble. Every now and then, however, the system can get itself into a state where it is is out of memory and it must write some pages to disk before any more memory can be allocated. If the act of writing those pages, itself, requires memory allocations, the system can deadlock. Systems with complicated block I/O setups - those using the device mapper, network-based storage, user-space filesystems, etc. - are the most susceptible to this problem. 

There has been a steady stream of patches aimed at solving this problem; the [write throttling patch](<http://lwn.net/Articles/245600/>) discussed here last August is one of them. The problem is inherently hard to solve, though; it looks like it may be with us for a long time. Or maybe not, if Daniel Phillips's new and rather aggressively promoted [writeout throttling patch](<http://lwn.net/Articles/261271/>) lives up to its hype. 

Daniel's patch is quite simple at its core. His approach for eliminating writeout-related deadlocks comes down to this: 

  * Establish a memory reserve from which (only) code performing writeout can allocate pages. In fact, this reserve already exists, in that some memory is reserved for the use of processes marked with the `PF_MEMALLOC` flag. 

  * Place an upper limit on the amount of memory which can be used for writeout to each device at any given time. 

The patch does not try to directly track the amount of memory which will be used by each writeout request; instead, it tasks block-level drivers with accounting for the number of "units" which will be used. To that end, it adds an `atomic_t` variable (called `available`) and a function pointer (`metric()`) to each request queue. When an outgoing request finds its way to `__generic_make_request()`, it is passed to `metric()` to get an estimate of the amount of resource which will be required to handle that request. If the estimated resource requirement exceeds the value of `available`, the process will simply block until a request completes and `available` is incremented to a sufficiently high level. 

The `metric()` function is to be supplied by the highest-level block driver responsible for the request queue. If that block driver is, itself, responsible for getting the data to the physical media, estimating the resource requirements will be relatively easy. The deadlock problems, however, tend to come up when I/O requests have to go through multiple layers of drivers; imagine a RAID array built on top of network-based storage devices, for example. In that case the top level will have to get resource requirement estimates from the lower levels, a problem which has not been addressed in this patch set. 

Andrew Morton [suggested an alternative approach](<https://lwn.net/Articles/261633/>) wherein the actual memory use by each block device would be tracked. A few hooks into the page allocation code would give a reasonable estimate of how much memory is dedicated to outstanding I/O requests at any given time; these hooks could also be used to make a guess at how much memory each new request can be expected to need. Then, the block layer could use that guess and the current usage to ensure that the device does not exceed its maximum allowable memory usage. Daniel eventually [rejected](<https://lwn.net/Articles/261634/>) this approach, saying that looking at current memory use is risky. It may well be that a given device is committed to serving I/O requests which will, before they are done, require quite a bit more memory than has been allocated so far. In that case, memory usage could eventually exceed the cap in a big way. It's better, says Daniel, to do a conservative accounting at the beginning. 

The patch does not address the memory reserve issue at all; instead, it relies on the current `PF_MEMALLOC` mechanism. It was necessary, says Daniel, to give the `PF_MEMALLOC` "privilege" to some system processes which assist in the writeout process, but nothing more than that was needed. He also claims that, for best results, much of the current code aimed at preventing writeout deadlocks needs to be removed from the kernel. He concludes: 

Let me close with perhaps the most relevant remarks: the attached code has been in heavy testing and in production for months now. Thus there is nothing theoretical when I say it works, and the patch speaks for itself in terms of obvious correctness. What I hope to add to this in the not too distant future is the news that we have removed hundreds of lines of existing kernel code, maintaining stability and improving performance. 

Since then, a couple of reviewers have pointed out problems in the code, dimming its aura of obvious correctness slightly. But nobody has found serious fault with the core idea. Determining its true effectiveness and making it work for a larger selection of storage configurations will take some time and effort. But, if the idea pans out, it could herald the end of a perennial and unpleasant problem for the Linux kernel.
