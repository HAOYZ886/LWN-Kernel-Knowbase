---
title: The next steps for swap
url: https://lwn.net/Articles/717707/
date: "March 22, 2017"
category: "Memory management-Swapping"
author: "By Jonathan Corbet March 22, 2017 LSFMM 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 22, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

Swapping has long been an unloved corner of the kernel's memory-management subsystem. As a general rule, the thinking went, if a system starts swapping the performance battle has already been lost, so there is little reason to try to optimize the performance of swapping itself. The growth of fast solid-state storage devices is changing that calculation, though, making swapping interesting again. At the 2017 Linux Storage, Filesystem, and Memory-Management Summit, Tim Chen led a session in the memory-management track that looked at the ways that swapping performance can be improved. 

Chen has been working on swapping performance for a while; [the first set of swap scalability patches](<https://lwn.net/Articles/704478/>) has already been merged. His next priority is improving swap readahead performance. This mechanism, which tries to read pages from swap ahead of an anticipated need for them, currently reads pages back in the order in which they were swapped out. This, he noted, is not necessarily the best order and, with mixed access patterns, performance can be poor. 

The recently submitted [VMA-based swap readahead patches](<https://lwn.net/Articles/716296/>) try to improve readahead performance by watching the swap-in behavior of each virtual memory area (VMA). If it appears that memory is being accessed in a serial fashion, the readahead window is increased in the hope of bringing in more pages before they are needed. For random patterns, instead, readahead has little value, so the window is reduced. 

Rik van Riel noted that the current readahead algorithm was designed for rotational media and asked how well the VMA-based mechanism works on such devices. Chen, with visible embarrassment, said that this hasn't been tried. Van Riel added that, with rotational devices, a group of adjacent [![\[Tim Chen\]](https://static.lwn.net/images/conf/2017/lsfmm/TimChen-sm.jpg)](<https://lwn.net/Articles/717766/>) blocks can be read as quickly as a single block can, so it makes sense to speculatively read extra data. The same is not true for solid-state storage. So, he suggested, it might make sense for the readahead code to see which type of device is hosting the swap space and change its behavior accordingly. 

Matthew Wilcox, instead, said that the real problem might be at swap-out time. Pages are swapped based on their position on the least-recently-used (LRU) lists, which may not reflect the order in which they will be needed again. He said that, perhaps, writes to swap could be buffered; swapped pages would go into a "victim cache" and sorted before being written to storage. The value of this approach wasn't clear to everybody in the room, though, given that access patterns can change over time. 

The next subject was the swapping of transparent huge pages. Currently, the first step is to split those pages into their component single pages, then to write those to swap individually — not the most efficient way to go about things. Chen and company would like to improve this behavior in a few steps, the first of which is to delay the splitting of the page until space has been allocated in the swap area. That should result in the allocation of a single cluster of pages for the entire huge page, at which point the whole thing can be written in a single operation. [Patches](<https://lwn.net/Articles/702159/>) implementing this change have been submitted; they result in a 14% swap-out performance increase. 

The next step is to delay the splitting of huge pages further, until the swap-out operation is finished. Those patches are in development; benchmarking shows that they result in a 37% improvement in swap-out performance. 

Finally, it would be nice to be able to swap huge pages directly back in. This idea needs more thought, he said. It is not always a performance win; if the application only needs a couple of small pages of data, there is no point in bringing in the whole huge page. One possible heuristic could be to only swap in huge pages for memory regions marked with `MADV_HUGEPAGE` or which have a large readahead window. 

There was a bit of discussion on how to justify the inclusion of these patches once they are ready. The best motivator is good benchmark results. It was suggested that Linus Torvalds is less likely to block the patches if they do not slow down kernel builds. Michal Hocko said that the patches were interesting, but that they were optimizing a rare event; the current code assumes that we don't ever want to swap. But Johannes Weiner said that the swap-out changes, at least, make a lot of sense; batching operations by keeping huge pages together will speed things up. 

The next topic was the use of the DAX direct-access mechanism with swapped data. If swapping is done to a persistent memory array, the data can still be accessed directly without the need to read it back into RAM. There is "an almost-working prototype" that does this, Chen said. The hard part is deciding when it makes sense to bring pages back into RAM; memory that will be frequently accessed, especially if the accesses are writes, is better read back in. 

Wilcox said that the decision really depends on the performance difference between dynamic RAM and persistent memory on the system in question; in some cases, the right answer might be "never". Sometimes, for example, the "persistent-memory array" is actually dynamic RAM hosted in a hypervisor. There was some talk of using the system's performance-monitoring unit (PMU) to track page accesses, but that idea didn't get far. Developers prefer that the kernel not take over the PMU, the runtime cost is high, and the results are not always all that useful. 

After some discussion, the conclusion reached was that the kernel should just bring a random set of pages back into RAM occasionally. With luck, the frequently used pages will stay there, while the rest will age back out to swap. 

Finally, there was a brief discussion of further optimizing the swap-device locking, which still sees significant contention even after the recent scalability improvements. So there is some interest in using [lock elision](<https://lwn.net/Articles/572099/#elision>) toward this end.
