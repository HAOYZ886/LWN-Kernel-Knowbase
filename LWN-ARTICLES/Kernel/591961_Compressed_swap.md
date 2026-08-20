---
title: Compressed swap
url: https://lwn.net/Articles/591961/
date: "March 26, 2014"
category: "Memory management-Swapping; Transcendent memory; zswap"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

There are a number of projects oriented around improving memory utilization through the compression of memory contents. Two of these, zswap and zram, have found their way into the mainline kernel; they both aim to replace swapping with compressed, in-memory storage of data. They differ in an important way, though. Zram acts like a special block device which can be mounted as a swap device; zswap, instead, uses the "[frontswap](<https://lwn.net/Articles/386090/>)" hooks to try to avoid swapping altogether. 

Bob Liu led a session to talk about this technology with a specific focus on zswap and a performance problem he has encountered with it. Zswap stores "swapped" data by compressing it and placing the result in a special RAM zone maintained by the "zbud" memory allocator. When the zbud pool fills, zswap must respond by evicting pages from that area and pushing them out to a real swap device. That involves decompressing the data, then writing the resulting pages to the swap device. That can slow things down significantly. 

Bob had a couple of options that he asked the group to consider. One of those was to turn zswap into a write-through cache; any pages stored in zswap would also be written to the swap device at the same time. That would allow the instant eviction of pages from zswap; since they already exist on the swap device, no further effort would be required. The cost, of course, would be in the form of increased swap device I/O and space usage. 

The second option would be to make the zswap area dynamic in size. It is currently a fixed-size region of memory. If it were dynamic, it could grow in response to increased demand. Of course, there would be limits to that growth, after which it would still be necessary to evict pages from the zswap area. 

Bob may have hoped for guidance with regard to which direction he should take, but he did not get it. Instead, Mel Gorman made the point that neither zram nor zswap has been well analyzed to quantify the benefits they provide to a running system. When people do run benchmarks, they tend to choose tests like [SPECjbb](<https://www.spec.org/jbb2013/>) which, he said, is not well suited to the job. Or they pick kernel compiles, which is even worse. 

What the compressed swapping subsystems really need, he said, is better demonstration workloads. In fact, they need those workloads so badly that no changes to the behavior of these subsystems will be considered until those workloads have been provided. So the real next step for developers working with compressed swapping is not to worry about how the system responds to pool exhaustion — at least, not until a better way to quantify the performance impact of any changes has been found. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
