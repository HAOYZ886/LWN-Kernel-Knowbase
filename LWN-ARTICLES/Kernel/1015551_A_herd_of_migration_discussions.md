---
title: A herd of migration discussions
url: https://lwn.net/Articles/1015551/
date: "March 31, 2025"
category: Memory management
author: "By Jonathan Corbet March 31, 2025 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 31, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Migration is the act of moving data from one location in physical memory to another. The kernel may migrate pages for many reasons, including defragmentation, improving NUMA locality, moving data to or from memory hosted on a peripheral device, or freeing a range of memory for other uses. Given the importance of migration to the memory-management subsystem, there is a lot of interest in improving its performance and removing impediments to its success. Several sessions in the memory-management track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit were dedicated to this topic. 

#### Migration with multi-threading

The first of those discussions, run by Zi Yan and (remotely) Shivank Garg, was focused on improving migration performance. Garg started by pointing out that migration is critical for optimal performance on both NUMA and tiered-memory systems. Yan, instead, is concerned with the migration of pages to and from device memory, where the devices involved can be GPUs or AI accelerators. Both would like migration to happen more efficiently and with less impact on system performance. 

Garg said that, in current kernels, migration speed is bounded by what single-threaded, CPU-bound operations can deliver. But many systems have hardware accelerators installed. Once upon a time, there might have been interest in using those accelerators to perform migration; that interest still exists, but we live in a different world now. On at least some of these systems, the real work is being done on the accelerators, while the CPUs mostly sit idle waiting for that work to be done. So it makes sense to dedicate _more_ CPU time to the migration task. 

One way to improve things is to batch the copying of pages (or, more correctly, folios) to their new location; that tends to improve the use of the available memory bandwidth. Better, though, is to perform a multi-threaded copy using multiple CPUs in parallel. Choosing the right number of threads can be tricky, though; more is not always better. The optimal number of threads is, naturally, architecture-specific. The placement of the copying operations also affects performance; using multiple CPUs will not make things any faster if they are all part of the same cache domain. The best solution in the end might involve the creation of a [sched_ext module](<https://lwn.net/Articles/922405/>) to manage placement. 

David Hildenbrand said that the batching of page migration could delay the execution of applications and asked whether any measurements had been done; Yan answered that these tests have not been run. But the way the batching is done, he said, does not increase latency since the page data is copied first, before the metadata. There can be some latency increases when copying 1GB pages, though, so he agreed that it is better to use a smaller batch size when there is a lot of concurrent access to the data. 

Ryan Roberts reiterated that the limiting factor currently is the memory bandwidth available to a single CPU; the changes being discussed work around that by using multiple CPUs. He wondered if some of the recently added Arm instructions could help here; another participant answered that the potential is there, but improvements have not been observed on actual hardware yet. 

Garg concluded the session with a brief discussion of offloading the migration copying to hardware on the system, with a fallback to CPU-based copying of the offloaded operation is not successful. This approach is implemented as a copy-offload driver, with static calls used to select the best approach at boot time. But the performance of this driver is not as high as expected, he said, due to the overhead of setting up the DMA mappings. Jason Gunthorpe was surprised by that, saying that mapping overhead should not be a problem. At the end, Zi reiterated that the goal of all of this work is to get better migration bandwidth. 

#### Migrating the unmovable

Not all pages (or folios) can be migrated; migration is only possible if that memory can be isolated (guaranteeing that nothing is trying to access it) and any users will still find it correctly after the move. User-space pages are usually movable, since they are accessed by way of virtual addresses; all that is needed is to change the page-table entries to point to the new location. The kernel tries hard to separate movable and unmovable memory so that one unmovable page doesn't block an attempt to clear a larger region. But sometimes an ostensibly movable page turns out not to be; Hildenbrand ran a session on how that situation comes about and what might be done about it. 

[![\[David
Hildenbrand\]](https://static.lwn.net/images/conf/2025/lsfmm/DavidHildenbrand-sm.png)](<https://lwn.net/Articles/1015655/>) He started by listing the various ways in which the system depends on successful migration. [Memory compaction](<https://lwn.net/Articles/368869/>) and defragmentation require the ability to move pages around, for example. NUMA balancing and tiering, the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>), the implementation of memory policies, CPU hotplugging, and device-private memory all need migration to work. Often, all it takes is a single unmovable page to create problems. 

Certain kinds of pages are inherently unmovable, he said; these include slab pages, page tables, and just about anything else that is not mapped into user space. On the other hand, most folios are movable, as is balloon memory in virtual machines, zswap storage, or anything allocated explicitly with the `__GFP_MOVABLE` flag. The goal is to have most pages be movable to give the system as much flexibility as possible. 

But, sometimes, pages that are supposed to be movable become fixed in place, leading to a number of problems, including CMA allocation failures, device-private memory being stuck in place, memory-hotplugging failures, and memory fragmentation. This lack of mobility can be either temporary or permanent, with different results. 

One cause of temporarily unmovable pages is simply running out of memory; if there is no place to move a page _to_ , it cannot be moved. A variant of that is caused by fragmentation; a large folio cannot be moved intact if the destination folio cannot be allocated. Splitting a large folio in this case is an option, but some large folios, such as dirty folios backed by an XFS filesystem, cannot be split for migration, and are thus unmovable. These problems usually resolve themselves in time. 

There are also problems with the LRU cache, which doesn't handle large folios (which add additional references to their pages); Matthew Wilcox suggested that perhaps that shortcoming should be fixed. Rik van Riel said that the LRU lock can be a bottleneck, so it is better to keep larger folios on the LRU to reduce contention. That, though, seemingly requires a redesign of the LRU; Michal Hocko said that this project is worth looking into, but that it would be a hard one. 

The most serious problem with temporarily unmovable pages is caused by short-term pinning of pages. When user space initiates a direct-I/O operation, for example, the pages involved must be pinned down; if the application maintains a constant stream of I/O, that pin will never be released and the pages become, in effect, permanently unmovable. Hildenbrand suggested that, when a page block has been marked for isolation (the removal of mappings so that the contents can be migrated), any subsequently requested direct-I/O operations should be delayed until the migration is complete. 

Permanently unmovable pages can be created by long-term pinning; this can happen, for example, with user-space memory that is used as an RDMA buffer. The kernel tries to migrate such memory out of the movable page blocks before allowing this kind of pin, but the migration can fail. Migration is not a perfect solution in any case; the pages may become movable again in short order, but now they will be occupying relatively expensive unmovable zones. Perhaps, Hildenbrand said, there should be a way for user space to warn the kernel that a range of memory will be long-term pinned; that would have to be implemented carefully, though. 

There are also special types of folios that can be permanently unmovable, including [secret memory](<https://lwn.net/Articles/865256/>) or [`guest_memfd()` areas](<https://lwn.net/Articles/949277/>). The kernel tries to keep these folios out of movable zones, and that code is mostly working as intended. Pages that have been marked as failed by the [HWPOISON subsystem](<https://lwn.net/Articles/348886/>) are unmovable, but are not normally a problem. Another source of unmovable pages is bugs in the kernel; [`vmsplice()`](<https://man7.org/linux/man-pages/man2/vmsplice.2.html>) can still pin pages forever, for example. 

The last problem that Hildenbrand raised before the session ran out of time was "infinite I/O". Pages that are under readahead or writeback are not movable, and that I/O can take forever. This can happen as the result of I/O errors, connectivity problems in network filesystems, or in FUSE filesystems. The last cause is the most serious, since it can be created by an unprivileged user-space process that simply never gets around to completing an I/O operation; there perhaps needs to be a way for the kernel to ask FUSE to cancel an operation, he said. 

Wilcox pointed out that, with network filesystems, the pages involved will be mapped for DMA I/O, which could happen at any time. There just is no way to free the memory without a revocation protocol for the specific device involved, and those protocols don't exist. Van Riel pointed out that these buffers have to be in movable memory, since that is most of the memory in the system. Hildenbrand closed by saying that solving this problem would clearly require a longer discussion at some other time. 

([The slides from this session](<https://drive.google.com/file/d/1uX80M1x86Oz3DFoHif-JLx1rlC_Nh93R/view>) are available.) 

#### Non-folio migration

Hildenbrand was not done, though; the next session addressed a different migration-related problem. Currently, migration is designed to move folios around in memory. That works now, since any non-tail page is by definition a folio. In the future, though, folios will only be one of many page types, and there is a strong desire to reduce the memory overhead of tracking those other page types. How can page migration work when the migration code no longer has access to the fields of `struct folio`? 

He started by saying that ""[frozen pages](<https://lwn.net/Articles/1000654/>) are great"", since they no longer use the reference-count field found in `struct folio` (and `struct page`). Any page type that can be made frozen has thus already taken an important step toward moving away from using folio-specific fields. He wanted to use the frozen-page concept for types of pages that nobody should touch — "logically offline" pages. These include pages in balloon drivers that are currently not in use. But the balloon drivers currently depend on the reference count, so they cannot use frozen pages. 

In a future where balloon pages are not folios, the balloon driver will have to stop using the reference count. But the problem is worse, in that other folio fields are used by migration; these include the `lru` field, which is a doubly linked list head. Either migration will have to be weaned off those fields, or every page in the system will have to provide them. That would mean allocating [memory descriptors](<https://lwn.net/Articles/974937/>) for all movable pages, which nobody really wants to do. 

At the lowest levels, the migration task is handled by [`struct movable_operations`](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/migrate.h#L23>): 

```
struct movable_operations {
    	bool (*isolate_page)(struct page *, isolate_mode_t);
    	int (*migrate_page)(struct page *dst, struct page *src, enum migrate_mode);
    	void (*putback_page)(struct page *);
        };
```

The `isolate_page()` callback attempts to remove all references to a page so it can be moved, `migrate_page()` actually moves it, and `putback_page()` will only be called if the migration fails and the page should go back into use in its original location. These operation do not need to depend on the folio fields, so it should be possible to manage migration for non-folio pages. 

Hildenbrand is working on a scheme to accomplish that. It is intended to work in the planned future where `struct page` has been replaced by a single, 64-bit memory descriptor. That descriptor may be a pointer to a larger structure, but it is hoped that, for many types of memory, the descriptor itself will be able to hold the needed information. Hildenbrand is specifically targeting those types of memory. His planned solution will avoid any folio-specific fields, meaning it is limited to whatever can be fit into the memory descriptor itself; Wilcox told him that there are 60 bits available for that purpose. 

Currently, the migration code uses the `mapping` folio field to store a pointer to the relevant `movable_operations` structure. Eliminating that will be easy, Hildenbrand said; the relevant operations will be found by using the page type instead. The page flags used to track the isolation of pages should not be needed, since the driver managing those pages should know if they can be (and have been) isolated. There will be no need for the reference count; the `isolate_page()` and `putback_page()` callbacks will transfer ownership of the page, while `migrate_page()` leaves the source page unreferenced. And, rather than using the `lru` field to store lists of pages to migrate, the migration code would simply keep an array of the pages it is working on. 

So far, this scheme is no more than that; Hildenbrand admitted that he did not have any code yet. Wilcox said that the plan looks good, though, and should solve a lot of problems. Hildenbrand hoped that this approach could be used to make more page types movable in the future; page-table-entry pages, for example, occupy a lot of memory but are not movable now. 

As a sort of coda to the conversation, Hildenbrand returned to balloon drivers, which store inflated pages in a linked list using the `lru` field. He suggested that a bitmask could be used to track those pages instead. The [ID allocation (IDA) subsystem](<https://docs.kernel.org/core-api/idr.html>) could be used to manage that bitmap, but it would need a few improvements first. 

([The slides for this presentation](<https://drive.google.com/file/d/1NIjjwVAonz9WWIoJZ0nh71ovMfSjfqFc/view>) are available as well.)
