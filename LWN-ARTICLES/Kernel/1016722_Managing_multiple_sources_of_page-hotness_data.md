---
title: "Managing multiple sources of page-hotness data"
url: https://lwn.net/Articles/1016722/
date: "April 11, 2025"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet April 11, 2025 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 11, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Knowing how frequently accessed a page of memory is (its "hotness") is a key input to many memory-management heuristics. Jonathan Cameron, in a memory-management track at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, pointed out that the number of sources of that kind of data is growing over time. He wanted to explore the questions of what commonality exists between data from those sources, and whether it makes sense to aggregate them all somehow. 

Cameron's own [focus](<https://lwn.net/Articles/998964/>) is on the [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>) "hotness monitoring unit", which can provide detailed data on which pages in a CXL memory bank have been accessed, but there are many other data sources as well. He fears that it may be crazy to try to combine them all, but hopes that it makes sense to do so at least some of the time. 

[![\[Jonathan
Cameron\]](https://static.lwn.net/images/conf/2025/lsfmm/JonathanCameron-sm.png)](<https://lwn.net/Articles/1016723/>) Hotness, he said, is a proxy for something that cannot be measured ahead of time: the performance cost of putting data in the wrong place. This data will drive actions like promotion, demotion, swapping, or reclaim. Different sources give different data, which may or may not include a virtual address, the NUMA node that made the access, or when the access happened. 

Combining this information from different providers can be challenging. The hotness monitoring unit, for example, does not see accesses that are resolved from the CPU's caches; that results in data that works reasonably well for tiering, but less so for the management of least-recently-used (LRU) lists. CPU-based hardware-supported methods do not include accesses from prefetch operations, making it harder to use that data to balance memory bandwidth use. Every way we have of measuring hotness, he said, is wrong in some way. 

Cameron mentioned the aggregation and promotion threads proposed by Bharata Rao, which were [discussed earlier in the day](<https://lwn.net/Articles/1016519/>). Rao, over the remote link, said that he was working to accumulate information from different sources and to provide an API that various subsystems could use. Cameron said that it was necessary to start with something, and that Rao's proposal seemed as good as any. 

Getting back into the meaning of "hotness", Cameron said that it is a guess at the future cost of moving (or not moving) memory. The kernel cannot see into the future, and cannot measure past costs, but it can measure access frequency to some extent. Some measurement methods, though, sample infrequently and can miss accesses. Techniques like access-bit scanning or tracking page faults can capture individual events; there is some commonality between sources of this type. What is needed is an efficient data structure to aggregate the data they produce. 

The hotness monitoring unit, though, only provides lists of hot pages, separated from the events that were observed. Data from all these sources goes into some sort of "hot list", he said, that may be used for page promotion. But the size of the hot list is constrained, he said, so the kernel cannot track everything. The entries in that list, as a result, are a sort of random subset of the hot pages, and the list can change frequently. There may be ways to improve this data, perhaps by tracking pages that were previously considered hot, but any solution will be complex and involve a lot of heuristics. 

Davidlohr Bueso asked if the aggregation of this data could be centralized in the DAMON subsystem. It has the APIs to do this aggregation and could be a good place to try things out to see what works. Cameron agreed, saying that there are a lot of good features in DAMON, but that its regions abstraction does not work well for some use cases. In the worst case, it can end up with each page in its own region. 

Cameron concluded by saying that it may be a while yet before the community knows how to unify these data sources; Bueso answered that it would be better to do something now to learn what works. Gregory Price asked for an asynchronous bulk page migrator that would run in its own thread. Making hotness data available in an API would help to make that happen, he said, and to evaluate the usefulness of each of those sources.
