---
title: "Memory-management scalability"
url: https://lwn.net/Articles/636334/
date: "March 13, 2015"
category: "Memory management-Scalability; Memory management-mmap sem; Scalability"
author: "By Jonathan Corbet March 13, 2015 LSFMM 2015"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 13, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

One of the drivers of memory-management development is scalability — performing well on ever-larger systems. So it is not surprising that scalability is a perennial discussion topic at kernel development gatherings; the 2015 Linux Storage, Filesystem, and Memory Management Summit was no exception. Andi Kleen and Peter Zijlstra led the first of two sessions on virtual memory scalability during the memory-management track at that event. 

Andi started by pointing out that systems were growing, not only in the number of CPU cores available, but also in the amount of attached memory. The number of cores per NUMA node is on the rise, which is bringing out some new scalability problems. 

One of the well-used scalability tactics found in the kernel is per-CPU variables; when each CPU has its own data, there can be no contention between them. But, Andi asserted, as the number of CPUs grows, it no longer makes [![\[Andi Kleen\]](https://static.lwn.net/images/conf/2015/lsfmm/AndiKleen-sm.jpg)](<https://lwn.net/Articles/636538/>) sense to do things on a per-CPU basis. It just adds a lot of work whenever it becomes necessary to touch every CPU's version of a variable. Instead, data should be made local to groups of N cores (where N was not specified). 

Christoph Lameter said that a lot of these scaling problems can be addressed by limiting subsystems to specific cores. Andi replied that this approach works great at installations where there is an experienced person configuring the system. In the absence of that person, it does not work quite so well. 

Mel Gorman asked the group what other scalability problems are being experienced now. Christoph complained about I/O bandwidth; in particular, he said, he is unable to push more than about 2GB/second to a filesystem. The problems come down to locking and the handling of 4KB pages in the XFS filesystem. Writeback tends to slow things down, since a lot of CPU time is spent making it happen. 

That led to a discussion of batching operations — another tried-and-true scalability technique. It was noted that the reverse-mapping code, which maintains data structures to enable the kernel to tell which processes have references to a given physical page, takes its locks on a per-page basis. Fixing that, evidently, is not hard, but it will require some reorganization of the code. 

The current least-recently-used (LRU) lists track memory in units of 4KB pages. That is considered at this point to be overly fine-grained; there is no need for LRU accuracy at that level. There was talk of implementing a "bucket LRU" that would track larger groups of pages. 

Inter-processor interrupts (IPIs) for translation lookaside buffer flushes have long been seen as a potential scalability problem. But, it seems [![\[Peter Zijlstra\]](https://static.lwn.net/images/conf/2015/lsfmm/PeterZijlstra-sm.jpg)](<https://lwn.net/Articles/636539/>) that, while people worry about IPIs, it is hard to find a workload where they create a bottleneck. Usually the much-maligned `mmap_sem` semaphore gets in the way first. 

There was some vague talk of other scalability issues; memory compaction was mentioned as a problem on large systems. If compaction tries to migrate a lot of pages, that can lead to large latencies in process execution. Mel Gorman said that compaction shouldn't be doing that, though, so it is not clear where the problem is. 

The session wound down without coming to any real conclusions. The scalability topic returned on the second day, though, when Davidlohr Bueso led a session focused on `mmap_sem` in particular. This semaphore controls access to a process's page tables, along with a number of other, not always well-defined things; it has been on the list of things to fix for some time now. Davidlohr stated a wish to walk out with some tangible action items for improving the situation. 

He started by looking back at past action items, especially those that came out of the [LSFMM 2014 locking session](<https://lwn.net/Articles/591978/>). One of the concerns then was use of `mmap_sem` in drivers and other code outside of the memory-management subsystem. Jan Kara has been working on getting drivers to use the `gup_fast()` variant of `get_user_pages()` in order to eliminate dependencies on `mmap_sem`; the biggest problem he is facing at the moment is a deadlock problem in the media subsystem. 

Jan would also like to get `mmap_sem` out of the filesystem code. Al Viro wondered, though, about how virtual memory area (VMA) structures would be protected in its absence. Peter said he has a patch that shifts the protection of VMAs to [sleepable RCU](<https://lwn.net/Articles/202847/>) if anybody wanted to push that work forward. Meanwhile, Jan hopes to get his driver patches submitted soon. 

Davidlohr said that his focus is moving stuff out from under `mmap_sem` entirely and, eventually, breaking up the lock into something finer-grained. The problem with that, as Peter pointed out, is [![\[Davidlohr Bueso\]](https://static.lwn.net/images/conf/2015/lsfmm/DavidlohrBueso-sm.jpg)](<https://lwn.net/Articles/636540/>) that what's protected by the lock now is not entirely clear. The way to start, he said, would be to document what's protected by `mmap_sem`; after that, one can start thinking about better locking schemes. 

One problem with `mmap_sem` is that it protects a process's entire address space. Concurrency could be increased by locking only portions of that space instead. The concept of "range locks" is thus of interest here. Michal Hocko suggested that developers could start by replacing `mmap_sem` with a range lock that still covers the entire address space; the locking could then be made more precise in an incremental manner. 

Hugh Dickins, though, wondered if that was the right approach and what problems, exactly, were being solved with range locks. His impression was that the top priority was to get page-fault handling out from under `mmap_sem` entirely. The answer was that there are, in fact, two different issues to be addressed regarding `mmap_sem`: it protects too much, and the hold times are too long. Range locks are one attempt to address the first part of the problem. Peter added that, among other things, range locking would allow concurrent `mmap()` calls to proceed, which is important for some threaded workloads. 

There was some concern about surprises that can pop up when it turns out that an unexpected corner of the code was relying on `mmap_sem`. In extreme cases, Hugh said, user-space code may even rely on it. He described a complaint from a user about a change in `mlock()` semantics. Changes in the kernel increased `mlock()` concurrency and, in the process, exposed a lack of locking on the user-space side. Sympathy for the affected user was relatively low in this case, but, Hugh said, it would be wise to be prepared for nasty surprises. 

In the end, Davidlohr's desire for tangible action items went mostly unfulfilled. About the only firm conclusion was that the range-lock code will be cleaned up and posted in the near future.
