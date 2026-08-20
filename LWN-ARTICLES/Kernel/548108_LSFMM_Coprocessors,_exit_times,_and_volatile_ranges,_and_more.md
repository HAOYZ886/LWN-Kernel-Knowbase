---
title: "LSFMM: Coprocessors, exit times, and volatile ranges, and more"
url: https://lwn.net/Articles/548108/
date: "April 23, 2013"
category: "Coprocessors; Filesystems-In user space; Memory management-Page replacement algorithms; Volatile ranges"
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Several sessions in the memory management track did not, for one reason or another, lend themselves to at-length reporting. For convenience, the notes from those sessions have been gathered together here. 

#### Hardware-initiated paging

Larry Woodman and Rik van Riel ran a session on the challenges posed by the increasingly capable coprocessors often found attached to contemporary systems. These coprocessors are getting fast, to the point that the nominal CPU is often one of the slower processors in the system. These processors have access to system memory, but they have their own memory management unit and their own page table format; they run software other than the Linux kernel. 

While coprocessors can access main memory directly, they have a couple of limitations that result from how they are connected: they may not be able to see _all_ of memory, and access to main memory is quite a bit slower than access to their own, local memory. So if a page of data is to be operated on extensively by a coprocessor, it makes sense to move that data to the coprocessor's memory. The implication is that there needs to be a way for the coprocessor to "fault" a page across the link, and for the host system to track which pages are currently resident in coprocessor memory. In a sense, those pages would look, to the host system, as if they had been swapped out. 

It was noted that the memory management features needed by coprocessors resemble the needs of remote DMA (RDMA), so it would be a good thing if they could both share the same framework. Andi Kleen added that there have been many attempts to incorporate high-speed coprocessors in the past; those technologies, he said, tend to look good on slides but to perform poorly in the real world. So, he suggested, we should not put a lot of effort into supporting those technologies now. 

That said, there does seem to be interest in adding support for coprocessors. Rik said that there is a reference implementation that is under development and which will, hopefully, be posted sometime this year. 

#### Process exit times and munlock()

Jörn Engel described a storage array product that he is working on. There are a number of interesting approaches being taken, but the root of his problem has to do with crash recovery. Should the array crash, it is supposed to produce a dump, reboot, and return to service within 30 seconds. But things are not working that way, mostly, he thought, as the result of the time to unwind the array's particular memory setup, which involves a lot of huge pages and pages locked into memory. 

The discussion wandered around for a while. In the end, it was concluded that Jörn was encountering a number of small memory management bugs that should be submitted to the mailing lists and dealt with there. 

#### Volatile ranges

John Stultz and Minchan Kim ran a session on volatile ranges, a subject that has been [covered here](<https://lwn.net/Kernel/Index/#Volatile_ranges>) several times in the past. The discussion touched on a number of relatively obscure technical topics, without much in the way of decisions being made. The best information on the state of the various volatile ranges patches can probably be found in [this detailed summary](<https://lwn.net/Articles/548111/>) sent out by John ahead of the event; for information about this session, see [the even more detailed report](<https://lwn.net/Articles/548151/>) John sent around. 

Hugh Dickins did ask why the virtual memory area (VMA) structure was not used to manage volatile ranges. The VMA, after all, is the structure used by the kernel to handle ranges of memory. The answer was that using VMAs would require acquiring the `mmap_sem` semaphore to make changes, and that is a cost that the developers wished to avoid. 

#### tmpfs and cpusets

Motohiro Kosaki was named as the leader of a session on memory power management. At the event, though, he stated that, while he had been asked to present the power management patches, he disagreed with them and could not lead a discussion about them. So he talked about problems in the interaction between the tmpfs filesystem, the cpuset feature, and memory policies. 

In particular, if you mount a tmpfs filesystem with the "`bind=relative`" memory policy (which attempts to perform memory allocations from NUMA nodes specified relative to the current node), kernel crashes can result. The problem seems to result from some sort of interaction with the cpuset feature, which binds processes to a specific set of processors. 

Andi responded that memory policies and cpusets are known to interoperate poorly; it would be best, he said, to just document the fact that the two features are not supported for use together. Cpusets, he said, have a sort of split function: they are used for both tuning and jailing processes to specific processors. The jailing part, he said, could probably just be dropped, since control groups do it better now. But, Ying Han objected, memory control groups do not currently understand NUMA locality, so they are not a complete replacement at this point. 

There are also evidently bad interactions between memory policies and memory hotplug; a hotplug event does not cause policies to be adjusted, leading to allocations from the wrong nodes. 

These issues were discussed for a bit, but no immediate solutions were forthcoming. 

#### Page cache reclaim with changing workloads

Johannes Weiner has noted a problem with how the memory management subsystem works: at times, it can fail to notice that the workload has shifted and that the pages in the "active" list are no longer active. That can lead to excessive reclaim of useful pages while the formerly-active pages that _should_ be reclaimed just sit there. This behavior is the result of a heuristic that is meant to prevent the active list from being pushed out of memory when a lot of paging is going on. 

The solution Johannes came up with is to improve the balancing of the active and inactive lists; this is done by keeping track of reclaimed pages to see how quickly they are being faulted back in. If the "refault distance" is less than the size of the active list, the code concludes that the active list is currently too big and shrinks its size by one page. This patch was [described fully](<https://lwn.net/Articles/495543/>) in May 2012, and hasn't changed much since then; Johannes brought it up here in the hope of getting some more substantial discussion. 

Michel Lespinasse questioned one aspect of the patch's behavior: when a "refaulted" page is detected, it is placed on the inactive list, along with one page from the active list. Since, he asked, a seemingly active page has been detected, why not put it directly onto the active list? The answer was that being put on the inactive list gave the page a bounded period of time to be accessed again and moved over to active status. It is, in a sense, being placed in competition with the page that was removed from the active list; whichever one is accessed first is more likely to make it back to active status. 

Johannes was also asked about the extra metadata that must be carried in the radix tree to track reclaimed pages. "Here comes reality," he responded. That data does tend to accumulate over time but, he said, it's only really a problem with gigantic files. The fact that the code throttles the tracking of reclaimed pages when refaults are not being detected helps as well, and there is the inevitable shrinker function to clean out old entries if need be. This seems like an area that may need a bit more work, but the patch as a whole, he said, performs well and should be considered for merging. 

#### FUSE writeback deadlocks

Maxim Patlasov recounted a problem with the FUSE (filesystems in user space) subsystem. When lots of pages on a FUSE filesystem are dirtied, the system fills with dirty pages to the point that things can lock up altogether. The kernel has a throttling mechanism that is supposed to place a limit on the number of dirty pages associated with each backing device, but that mechanism is not working with FUSE. 

The problem appears to be this: when FUSE-backed pages are dirtied, the throttling counters are incremented as usual. But when FUSE gets around to writing them back, it first allocates new pages for copies of the dirtied pages. This is, essentially, the FUSE solution to the "stable pages" problem. It works, but those new pages are not counted against the dirty page limit for the FUSE device, so user space is free to dirty even more pages. This process can go on indefinitely until there are no more pages to fill. 

The first solution that comes to mind is to count those temporary pages against the per-backing-device limit as well. But that approach, it seems, runs into a snag: it will throttle the FUSE daemon that is actively trying to clean those pages, making things worse. The solution is likely to take one of two forms: (1) add a process flag to mark the FUSE daemon and exempt it from the normal dirty limits, or (2) count FUSE-backed dirty pages twice to account for the number of pages actually required.
