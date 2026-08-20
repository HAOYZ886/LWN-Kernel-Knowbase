---
title: Fixing the contiguous memory allocator
url: https://lwn.net/Articles/636234/
date: "March 11, 2015"
category: "Memory management-Large allocations"
author: "By Jonathan Corbet March 11, 2015 LSFMM 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 11, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

Normally, kernel code goes far out of its way to avoid the need to allocate large, physically contiguous regions of memory, for a simple reason: the memory fragmentation that results as the system runs can make such regions hard to find. But some hardware requires these regions to operate properly; low-end camera devices are a common example. The kernel's [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>) exists to meet this need, but, as two sessions dedicated to CMA during the 2015 Linux Storage, Filesystem, and Memory Management Summit showed, there are a number of problems still to be worked out. 

CMA works by reserving a zone of memory for large allocations. But the device needing large buffers is probably not active at all times; keeping that memory idle when the device does not need it would be wasteful. So the memory-management code will allow other parts of the kernel to allocate memory from the CMA zone, but only if those allocations are marked as being movable. That allows the kernel to move things out of the way should the need for a large allocation arise. 

Laura Abbott started off the session by noting that there are a number of problems with CMA, relating to both the reliability of large allocations and the performance of the system as a whole. There are a couple of [![\[Laura Abbott\]](https://static.lwn.net/images/conf/2015/lsfmm/LauraAbbott-sm.jpg)](<https://lwn.net/Articles/636239/>) proposals out there to fix it — [guaranteed CMA](<https://lwn.net/Articles/619865/>) by SeongJae Park and [ZONE_CMA](<https://lwn.net/Articles/633101/>) from Joonsoo Kim — but no consensus on how to proceed. Joonsoo helped to lead the session, as did Gioh Kim. 

Peter Zijlstra asked for some details on what the specific problems are. A big one appears to be the presence of pinned pages in the CMA region. All it takes is one unmovable page to prevent the allocation of a large buffer, which is why pinned pages are not supposed to exist in the CMA area. It turns out that pages are sometimes allocated as movable, but then get pinned afterward. Many of these pins are relatively short-lived, but sometimes they can stay around for quite a while. Even relatively short-lived pins can be a problem, though; delaying the startup of a device like a camera can appear as an outright failure to the user. 

One particular offender, according to Gioh, appears to be the ext4 filesystem which, among other things, is putting superblocks (which are pinned for as long as the associated filesystem is mounted) in movable memory. Other code is doing similar things, though. The solution in these cases is relatively straightforward: find the erroneous code and fix it. The complication here, according to Hugh Dickins, is that a filesystem may not know that a page will need to be pinned at the time it is allocated. 

Mel Gorman suggested that, whenever a page changes state in a way that could block a CMA allocation, it should be migrated immediately. Even something as transient as pinning a dirty page for writeback could result in that page being shifted out of the CMA area. It would be relatively simple to put hooks into the memory-management code to do the necessary migrations. [![\[Joonsoo Kim\]](https://static.lwn.net/images/conf/2015/lsfmm/JoonsooKim-sm.jpg)](<https://lwn.net/Articles/636258/>) The various implementations of `get_user_pages()` would be one example; the page fault handler when a page is first dirtied would be another. A warning could be added when `get_page()` is called to pin a page in the CMA area to call attention to other problematic uses. This approach, it was thought, could help to avoid the need for more complex solutions within CMA itself. 

Of course, that sort of change could lead to lots of warning noise for cases when pages are pinned for extremely short periods of time. Peter suggested adding a variant of `get_page()` to annotate those cases. Dave Hansen suggested, instead, that `put_page()` could be instrumented to look at how long the page was pinned and issue warnings for excessive cases. 

The second class of problems has to do with insufficient utilization of the CMA area when the large buffers are not needed. Mel initially answered that CMA was simply meant to work that way and that it would not be possible to relax the constraints on the use of the CMA area without breaking it. It eventually became clear that the situation is a bit more subtle than that, but that had to wait until the second session on the following day. 

It took a while to get to the heart of the problem on the second day, but Joonsoo finally described it as something like the following. The memory-management code tries to avoid allocations from the CMA area entirely whenever possible. As soon as the non-CMA part of memory starts to fill, though, it becomes necessary to allocate movable pages from the CMA area. But, at that point, memory looks tight, so `kswapd` starts running and reclaiming memory. The newly reclaimed memory, probably being outside of the CMA area, will be preferentially used for new allocations. The end result is that memory in the CMA area goes mostly unused, even when the system is under memory pressure. 

Gioh talked about his use case, in which Linux is embedded in televisions. There is a limited amount of memory in a TV; some of it must be reserved for the processing of 3D or high-resolution streams. When that is not [![\[Gioh Kim\]](https://static.lwn.net/images/conf/2015/lsfmm/GiohKim-sm.jpg)](<https://lwn.net/Articles/636259/>) being done, though, it is important to be able to utilize that memory for other purposes. But the kernel is not making much use of that memory when it is available; this is just the problem described by Joonsoo. 

Joonsoo's solution involves adding a new zone (`ZONE_CMA`) to the memory-management subsystem. Moving the CMA area into a separate zone makes it relatively easy to adjust the policies for allocation from that area without, crucially, adding more hooks to the allocator's fast paths. But, as Mel said, there are disadvantages to this approach. Adding a zone will change how page aging is done, making it slower and more cache-intensive since there will be more lists to search. These costs will be paid only on systems where CMA is enabled so, he said, it is ultimately a CMA issue, but people should be aware that those costs will exist. That is the reason that a separate zone was not used for CMA from the beginning. 

Dave suggested combining `ZONE_CMA` with `ZONE_MOVABLE`, which is also meant for allocations that can be relocated on demand. The problem there, according to Joonsoo, is that memory in `ZONE_MOVABLE` can be taken offline, while memory for CMA should not be unpluggable in that way. Putting CMA memory into its own zone also makes it easier to control allocation policies and to create statistics on the utilization of CMA memory. 

The session ended with Mel noting that there did not appear to be any formal objections to the `ZONE_CMA` plan. But, he warned, the CMA developers, by going down that path, will be trading one set of problems for another. Since the tradeoff only affects CMA users, it will be up to them to decide whether it is worthwhile.
