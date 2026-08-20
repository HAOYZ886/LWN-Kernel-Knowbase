---
title: Fast memory allocation for networking
url: https://lwn.net/Articles/717767/
date: "March 22, 2017"
category: "Memory management-Scalability"
author: "By Jonathan Corbet March 22, 2017 LSFMM 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 22, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017>)

At the 2016 Linux Storage, Filesystem, and Memory-Management Summit, Jesper Dangaard Brouer [proposed](<https://lwn.net/Articles/684616/>) the creation of a new set of memory-allocation APIs to meet the performance needs of the networking stack. In 2017, he returned to the LSFMM memory-management track to update the community on the work that has been done in that area — and what still needs to be accomplished. 

Networking, he said, deals with "mind-boggling speeds"; a 10GB Ethernet link can handle up to nearly 15 million packets per second. On current hardware, that gives the operating system only about 200 processor cycles to deal with each packet. The problem gets worse as link speeds increase, of course. 

The main trick used in solving this problem is batching operations whenever possible. That is not a magic solution, he said; batching ten operations does not yield a 10x performance improvement. But it does help a lot and needs to be done. Furthermore, various kernel-bypass networking solutions [![\[Jesper Brouer\]](https://static.lwn.net/images/conf/2017/lsfmm/JesperBrouer-sm.jpg)](<https://lwn.net/Articles/717781/>) show that processing packets at these rates is possible; they work using batching and special memory allocators. They also use techniques like polling, which wastes CPU time; he thinks that the kernel can do better than that. 

One step that has been taken in that direction is the merging of the [express data path (XDP)](<https://lwn.net/Articles/682538/>) mechanism around the 4.9 development cycle. With XDP, it is possible to achieve full wire speeds in the kernel, but only if the memory-management layer is avoided. That means holding onto buffers, but also keeping them continually mapped for DMA operations. When that is done, a simple "drop every packet" application using XDP can handle 17 million packets per second, while an application that retransmits each packet through the same interface it arrived on can handle 10 million packets per second. These benchmarks may seem artificial, but they solve real-world problems: blocking denial-of-service attacks and load balancing. Facebook is currently using XDP for these tasks, he said. 

What has not been done with XDP so far is real packet forwarding, because that requires interactions with memory management. The page allocator is simply too slow, so current drivers work by recycling the pages they have allocated. Every high-performance driver has implemented some variant of this technique, he said. It would be good to move some of this functionality into common code. 

The general statement of the problem is that drivers want to get DMA-mapped pages and keep them around for multiple uses. The memory-management layer can help by providing faster per-CPU page caching ([some work](<https://lwn.net/Articles/711075/>) toward that goal was merged recently), but it still can't compete with simply recycling pages in the drivers. So he has another idea: create a per-device allocator for DMA-mapped pages with a limited cache. By keeping pages mapped for the device, this allocator could go a long way toward reducing memory-management costs. 

Matthew Wilcox asked if the existing DMA pool API could be used for this purpose. The problem, Brouer said, is that DMA pools are oriented toward coherent DMA operations (where long-lived buffers are accessed by both the CPU and the device), while networking uses streaming DMA operations (short-lived buffers that can only be accessed by one side or the other at any given time). 

What he really wants, Brouer continued, is to be able to provide a destructor callback that is invoked when a page's reference count drops to zero. That callback would be allowed to "steal" the page, keeping it available for use in the same driver. This callback mechanism actually exists now, but only for higher-order pages; bringing it to single pages would require finding room in the crowded `page` structure, which is not an easy task. Pages with destructors might also need a page flag to identify them, which is another problem; those flags [are in short supply](<https://lwn.net/Articles/335768/>). There was some discussion of tricks that could be employed (such as placing a sentinel value in the `mapping` field) to shoehorn the needed information into `struct page`; it seems likely that some kind of solution could be found. 

Brouer concluded with some benchmarks showing that the situation got better in the 4.11 kernel, thanks to the page-caching improvements done by Mel Gorman. But there is still a lot of overhead, much of which turns out to be in the maintenance of the zone statistics. These statistics are not needed for the operation of the memory-management subsystem itself, but it seems that quite a few users do make use of them to tune their systems. Gorman said that, when performance regresses, users typically report the problem within a release cycle or two, suggesting that they are indeed looking at the numbers. 

So the statistics need to remain, but it may be possible to disable their collection on production systems. The statistics code could probably be shorted out with a static branch in settings where they are not wanted. It is deemed worthwhile to run the benchmarks with NUMA disabled to see if any benefit is to be found there. 

At the end, Brouer asked whether there would be objections to a DMA-page pool mechanism. There were no immediate objections, but the developers in the room made it clear that they would want to see the patches before coming to any definite conclusions.
