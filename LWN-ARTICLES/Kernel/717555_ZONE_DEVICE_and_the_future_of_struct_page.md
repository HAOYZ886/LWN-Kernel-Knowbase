---
title: ZONE_DEVICE and the future of struct page
url: https://lwn.net/Articles/717555/
date: "March 21, 2017"
category: "Memory management-ZONE DEVICE"
author: "By Jonathan Corbet March 21, 2017 LSFMM 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 21, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

The opening session of the 2017 Linux Storage, Filesystem, and Memory-Management Summit covered [a familiar topic](<https://lwn.net/Articles/644079/>): how to represent (possibly massive) persistent-memory arrays to various subsystems in the kernel. This session, led by Dan Williams, focused in particular on the `ZONE_DEVICE` abstraction and whether the kernel should use `page` structures to represent persistent memory or not. 

`ZONE_DEVICE` is tied into the memory allocator's zone system (which segregates memory based on attributes like NUMA node or DMA reachability), but in a special way. It was created to satisfy the need to perform DMA operations on persistent memory; these operations require `page` structures to set up the mappings. `ZONE_DEVICE` is, he said, essentially the top half of the memory hotplug mechanism; it performs the memory setup, but does not actually put the pages online for general use. So memory located in `ZONE_DEVICE` cannot be allocated in the normal ways, pages cannot be migrated into that space, etc. But it is possible to get a `page` structure for memory in that zone. 

Over the past few years, as the development community looked at the implications of large persistent-memory arrays, developers were concerned about the cost of using `page` structures — 64 bytes for every [![\[Dan Williams\]](https://static.lwn.net/images/conf/2017/lsfmm/DanWilliams-sm.jpg)](<https://lwn.net/Articles/717618/>) 4KB memory page. That usage seemed wasteful, so some significant effort went into trying to avoid using `page` structures altogether; instead, it was thought that the management of persistent memory could be done entirely with page-frame numbers (PFNs). The [`pfn_t` type](<https://lwn.net/Articles/672457/>), along with a bunch of supporting structure, was added toward that goal, and developers tried to convert the entire DMA API to use PFNs. But then they ran into the SPARC64 architecture, which cannot create DMA mappings without using `page` structures. The `pfn_t` effort, Williams said, died there. 

Now, he said, perhaps the time has come to stop trying to avoid `struct page`. If, instead, we let drivers assume that `page` structures will be available, we'll pay the memory-use cost in systems with terabytes of persistent memory, but we'll avoid dealing with a lot of custom driver code with inconsistent behavior. That would solve the DMA problem, but that's probably the easiest of the problems in this area; `struct page` tends to pop up in a lot of places. 

Matthew Wilcox observed that, in truth, few drivers really care about `struct page` itself; it really just serves as a convenient handle for referring to physical memory. He suggested that it might make sense to go back and take a hard look at why SPARC is stuck with using `page` structures; Williams said it had to do with the management of cache aliasing state. James Bottomley suggested that there may be other ways to solve this problem, such as using a separate array to hold aliasing information. It would just be a matter of persuading SPARC maintainer Dave Miller. 

If that persuasion could be accomplished, then `pfn_t` could be used nearly everywhere and there would be less need to worry about the availability of `page` structures. A remaining problem might be drivers that need to reach directly into DMA buffers but, Wilcox said, they should just use `ioremap()` to get a usable address to work with. 

One of the big motivations for avoiding `struct page` with persistent-memory arrays is that these structures can end up filling a large portion of the system's ordinary memory. The way to avoid that, of course, is to allocate the structures in the persistent-memory itself; Wilcox said that, whenever new memory is added to the system, its associated page structures should always be located in that new memory. The problem is that `struct page` can be a heavily used structure, so there is value in having the ability to control its placement. 

One possible solution to the memory-use problem is to allow `page` structures to refer to larger pages — 2MB huge pages, for example. The problem here is that making the size variable would add overhead to some of the hottest code paths in the kernel. There would be CPU-time savings in some areas, since the number of `page` structures to be managed would be reduced considerably, but there are doubts that the savings would make up for the higher costs in places like the page allocator. 

Another option, Williams said, is to allocate `page` structures dynamically when they are needed. A persistent-memory array can be terabytes in size, but `page` structures may only be needed for a small portion of it. If allocation of `page` structures can be made cheap, it would make sense to only bring them into existence when the need arises. 

The conversation wound down in a wandering manner. Bottomley suggested using radix trees to track ranges of memory instead. Kirill Shutemov pointed out that different kinds of information are needed for different page sizes; in the case of transparent huge pages, it may be necessary to refer to a 4KB page as both a single page and a component of a huge page. Rik van Riel said that `page` structures are only really an issue for dynamic RAM; they can be dispensed with for persistent memory, since filesystems can be counted on to free memory when it's no longer in use. Bottomley replied that this approach is possible, but nobody has been willing to implement it so far, leading Williams to observe that the group would be talking about the same problem again next year.
