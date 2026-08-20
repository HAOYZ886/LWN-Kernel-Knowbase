---
title: "Bulk memory-allocation APIs"
url: https://lwn.net/Articles/684616/
date: "April 23, 2016"
category: "Networking-Performance"
author: "By Jonathan Corbet April 23, 2016 LSFMM 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 23, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

Jesper Dangaard Brouer has been [working on improving networking performance](<https://lwn.net/Articles/629155/>) for some time. When one is trying to process packets with a tight per-packet time budget, every bit of overhead counts, and the overhead of the memory allocator turns out to count quite a bit. At the 2016 Linux Filesystem, Storage, and Memory-Management Summit, he presented some ideas he had for reducing that overhead. 

One area of trouble is the DMA API, especially on PowerPC systems. There, the mapping and unmapping operations are expensive, and pages that have been mapped for DMA must be considered read-only by the networking code. Unfortunately, those pages must be written at times (to change an IP address, for example), leading to expensive unmapping operations that, perhaps, can even be destructive to the data in the buffer. 

Jesper suggested that, instead, a set of pages could be kept permanently mapped to the device and recycled when mapping requests are made. The pages could remain mapped, with the `dma_sync_*` functions used to control whether the device or the CPU "owns" the pages at any given time. That would cut out a lot of the overhead and speed up packet processing. One [![\[Jesper Brouer\]](https://static.lwn.net/images/conf/2016/lsfmm/JesperBrouer-sm.jpg)](<https://lwn.net/Articles/684621/>) little detail is that it would require space in the `page` structure to indicate which device a page is dedicated to; space in that structure tends to be in short supply. 

On x86 systems, DMA is not usually the problem; instead, memory allocation is. He is working with a target of 14.8 million packets (full wire speed on a 10Gb/s link) per second; on a 3GHz system, that gives the kernel about 200 cycles in which to process each packet. Allocating a single page, though, takes 277 cycles on that system, making the 14.8Mpps rate unattainable. He pointed out [Mel Gorman's recent work](<https://lwn.net/Articles/683988/>) to reduce memory-allocator overhead; that work reduced the cost to 230 cycles — a significant improvement, but nowhere near enough. 

The [DPDK](<http://dpdk.org/>) user-space networking code can achieve the 14.8Mpps rate, he said, so it is clear that the kernel is doing something wrong and not using the hardware optimally. After two years of optimization work, he has managed to double the kernel's processing rate to about 2Mpps, which, while being a step in the right direction, is far from the target. 

Drivers tend to work around the problem by allocating large pages and splitting them into pieces. An order-3 (eight-page) chunk can be allocated in 503 cycles, which is more than the single-page cost, but, when split into 4KB chunks, the per-page cost drops to 62 cycles. That clearly helps to reach the time budgets, but it has the disadvantage of pinning down large chunks of memory if some piece of code hangs on to just one of the component pages. That, in turn, can push the system as a whole into an out-of-memory situation, which can also adversely affect packet rates (among other things). For this reason, Google turns this mechanism off internally. 

Wouldn't it be better, he said, to just have a bulk page-allocation API that would return multiple pages without the need to allocate them as a single large page? Mel Gorman answered that, in the end, it was just a matter of coding to make that idea work. The page allocator already has the bulking concept internally, there just has been no reason to expose it to the rest of the kernel before. There should be no fundamental problem with doing so, he said. In general, there has not been a huge emphasis on page-allocation performance optimization because most users immediately zero each page after allocating it. The cost of clearing the page's contents overwhelms the cost of allocating the page in the first place. 

What worries Mel, though, is the idea of implementing an entirely new memory allocator. Eventually that allocator would have to gain features like NUMA awareness and would pick up "all the old mistakes," at which point it would be as complex and slow as what we have now. It would be better, he said, to use the existing per-CPU allocator and optimize it for this case; then all users in the kernel would benefit. He has a patch now that can halve the overhead of the page allocator if "certain assumptions are made"; in particular, the user needs to not care about NUMA placement and there should be a fair amount of memory available in general. 

Jesper returned to the idea of a "page pool" that would be used to quickly satisfy requests from network drivers (or other time-sensitive users). This pool would be small, the number of pages required would be about equal to the sum of the size of the receive ring and the transmit queue for each device. It is important to bound the size of the pool, or a persistent traffic overload can run the system out of memory. To that end, he said, the page allocator should be able to shrink the page pool when memory is tight. 

He saw two ways of creating this pool. One, that he called "all in," would be to write an entirely new slab-like allocator. The alternative is to wrap some sort of layer around the existing page allocator. Predictably, Mel was not fond of the "all in" strategy; he said we already have too many allocators and adding another will just create new problems. It would be better to add a bulk interface to the current allocator which, he repeated, could be made to be much faster in some settings. 

The direction for this work thus seems clear, but the challenges to overcome are significant. It may be a little while yet before a stock kernel can hit that 14.8Mpps networking target.
