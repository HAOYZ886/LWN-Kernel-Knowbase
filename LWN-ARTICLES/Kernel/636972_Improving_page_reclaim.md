---
title: Improving page reclaim
url: https://lwn.net/Articles/636972/
date: "March 17, 2015"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet March 17, 2015 LSFMM 2015"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 17, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

Dave Hansen started a brief LSFMM 2015 memory-management track session on page-reclaim performance by saying that we have a problem: over the years, the kernel's memory-management and swap subsystems have been designed around the use of slow secondary storage devices. But now we are heading toward an era increasingly driven by the availability of massive nonvolatile memory, and we are not fully prepared for it. 

The fundamental question, he said, was how to integrate these technologies into the Linux kernel. We have a number of subsystems like [DAX](<https://lwn.net/Articles/636096/>) that can provide high-speed access to persistent memory devices, but they require applications to be changed. If we run current kernels over such devices without using special interfaces, swapping is no faster than it is with older, slower devices. There is just too much overhead in the memory-management layer, and, in particular, in the manipulation of the least-recently-used (LRU) lists that track reclaimable pages in the system. The LRU, he said, is a fancy system to find the best eviction candidate at any given time, but, in this situation, perhaps it would be better to use something else? 

Christoph Lameter suggested that users who care about performance should just put their entire application into memory and be done with it. But Dave was not so easily deterred; he would like to find ways for existing applications to get better performance on persistent-memory devices without changes. 

Andrea Arcangeli said that we should not be worrying about memory in 4KB units when we are dealing with devices that can hold 100GB or more. Swapping pages in 2MB units would, he said, go a long way toward solving the problem. Andi Kleen agreed to a point — but he felt that 2MB was still far too small. In general, he said, we need to move toward managing memory in larger chunks or just do away with the LRU lists altogether. 

Dave suggested that there are a number of opportunities to run the LRU lists in a more relaxed mode. One idea, he said, was to add a third LRU level for pages that are ready to be swapped out. (The kernel currently manages two levels of LRU lists, one for active pages and one for pages that seem to be inactive and should be considered for eviction). Perhaps some sort of "scanaround" algorithm could be applied to that third level to batch up pages for writing out to the swap device. Johannes Weiner answered that he had tried something similar a few years ago. It didn't work well, he said, due to disk seek issues, but it might work better on truly random-access devices. 

Hugh Dickins expressed skepticism toward the entire idea, though. To him, it looks like an attempt to reduce memory-management overhead by adding even more complex algorithms to cluster things. That is increasing the complexity of the system rather than reducing it. Batching things up may help to speed things up, but you still have to deal with items individually to make up the batches. 

As things wound down, Dave said that he was going away with a couple of interesting ideas to explore.
