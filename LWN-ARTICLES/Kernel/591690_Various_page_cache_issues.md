---
title: Various page cache issues
url: https://lwn.net/Articles/591690/
date: "March 26, 2014"
category: "Memory management-Page cache; fsblock"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

The Linux page cache is charged with maintaining the cache of blocks of data from persistent storage devices; it is a fundamental component of both the virtual filesystem and memory management subsystems. The page cache is key to the system's performance as a whole. Unfortunately, it is showing its age in a number of ways. The first technical session at the 2014 Linux Storage, Filesystem and Memory Management Summit discussed a number of page-cache-related problems and mapped a rough path toward their solution. 

#### Large drives on 32-bit systems

One looming problem, introduced by James Bottomley, is that, on 32-bit systems, the page cache cannot work with drives larger than 16TB. The page cache uses the native page size (4KB) to address blocks on devices; with a 32-bit block index, the ability to represent blocks is exhausted at 16TB. The questions that James put to the group were: is this problem worth fixing on 32-bit systems, and, if so, how could be that done? 

A number of developers clearly felt that there was no need to solve this problem. They see 32-bit systems as being slowly on their way out; anybody who wants to use large storage devices, it follows, should just get a 64-bit system to run them on. In the real world, though, things might not be that easy. There will be 32-bit devices out there for some time, especially in the embedded world. People will get 16TB USB-connected devices and expect them to work. Others will put a 32-bit processor into a cheap network-attached storage device and want to put large drives into it. There are also manufacturers putting 32-bit processors directly onto drives to allow them to support protocols like iSCSI. 

As Dave Chinner pointed out, the core problem is in the page cache itself. If an application does direct I/O (which bypasses the page cache), there are no problems. Where things go bad is when user space attempts buffered I/O on a large device. That happens when filesystems are used, but also before that point: udev does buffered I/O to map out drives, for example. And there is one other little problem: even if a 16TB filesystem could be mounted and used on a 32-bit system, there is not enough address space to run a filesystem checker on it. That problem appears to be intractable on these systems. Even if the kernel is fixed, in other words, the rest of the ecosystem is just broken. 

These observations led to a rough consensus in the room. There will be no real attempt to make larger filesystems work on 32-bit systems. But the direct I/O path will work with very large drives and will remain supported. So use cases that depend only on direct I/O — the iSCSI drive case, for example — will work on 32-bit systems. For the rest, it will just be necessary to get a 64-bit CPU. 

#### Large pages in the page cache

Dave, along with Christoph Lameter, then moved the session onto the next topic: storing larger blocks of data in the page cache. It turned out that they want to do this for different reasons which are driving them toward different solutions. It may be possible to solve both problems, but one solution is likely to come sooner than the other. 

Christoph is worried about operating system overhead at all levels of the stack; part of his solution is to support larger physical pages in the page cache. Larger physical pages would reduce management overhead in the kernel and the amount of setup and teardown associated with I/O operations [![\[Christoph Lameter\]](https://static.lwn.net/images/conf/2014/lsfmm/ChristophLameter-sm.jpg)](<https://lwn.net/Articles/591836/>) on those pages. It may be possible to add an "order" attribute to page cache sizes, allowing differently sized (and larger) pages to be stored there, but there is trouble lurking in a familiar place: maintaining the ability to actually allocate large physical pages after the system has been running for a while. 

One solution, Christoph said, is to maintain a reserve of large pages for only this purpose, but that is "awkward" and hard to tune correctly. The other approach is to allow the kernel to move pages around, defragmenting things at run time. Much of the support is already there; the memory-management subsystem has the ability to migrate pages, and that ability is used in the [page compaction](<https://lwn.net/Articles/368869/>) code for memory defragmentation. The problem is that not all pages are movable; since it only takes one unmovable page to break up a large physical page, that is a significant roadblock. 

There are a number of reasons why a particular page might not be movable, but one of the most significant causes of unmovable pages is the allocation of kernel-space objects. If objects obtained from the slab allocators were to be movable, though, this problem would go away. Christoph has patches to add this functionality for some heavily-used caches (the inode cache, for example), but they have not been merged. There was some resistance to this idea; the kernel is full pointers to objects; no object can be moved unless all of those pointers can be changed without introducing race conditions. Christoph maintained that much of the problem could be solved by addressing a few important data structures. 

Even then, though, things may not work as well as Christoph might like. Mel Gorman made the point that, even on current systems where attempts are made to segregate movable and unmovable pages, a surprising amount of "movable" memory turns out not to be. Page table pages can be a substantial portion of memory, and they cannot be moved; a patch to fix that added 5-6% overhead, and was thus not merged. Other pages are allegedly movable but are pinned for direct I/O or otherwise locked in memory; even when a lot of care is taken, it can be hard to migrate pages. For now, the main symptom of this problem for most users is that transparent huge page allocations fail; that is not a huge problem. But, Mel said, if we come to really depend on being able to move pages, "it will blow up in our face." 

#### Larger blocks in filesystems

Dave had a related but different question to ask. He is interested in better supporting filesystems with block sizes that are larger than the system's page size. Supporting larger physically contiguous pages in the page cache could help toward that goal, and this approach offers some advantages, primarily being that it requires almost no changes in filesystem code to support. But the same cannot be said for the memory management subsystem, where some rather harder changes would need to be made. Rather than jump into all that work, he suggested, we should consider whether we really need to solve the problem that way. 

Part of the impetus toward larger physical pages has been limitations in the kernel's block I/O stack; the maximum size of a single I/O request was, for a long time, too small to get full performance out of high-speed storage devices. Raising the page size would raise that limit, allowing more data to be transferred with the same number of pages, fixing the problem. But the request size limit has long been solved, so there is much less need to solve the page-size problem now. [![\[Dave Chinner\]](https://static.lwn.net/images/conf/2014/lsfmm/DaveChinner-sm.jpg)](<https://lwn.net/Articles/591837/>) The real problem, he said, is that the page cache knows about the block sizes used within each filesystem. It ends up tracking a lot of filesystem-level information, duplicating the information stored in the filesystems themselves. This duplication leads to coherency issues and occasional nasty bugs; it is also the source of the limit that forces filesystem block sizes to be no larger than the system page size. 

Nick Piggin tried to solve this problem some years ago with his [fsblock](<https://lwn.net/Articles/322668/>) work. The problem with that patch set is that it required extensive changes to every filesystem. Even the ext2 conversion, done as a proof of concept by Nick, needed a lot of changes. So Dave has been looking at a different approach. In short, he would like the page cache to stop maintaining a mapping between pages and filesystem blocks; it would, instead, concern itself with the state of the pages themselves. Everything else is already known in the filesystems; getting that tracking out of the page cache would make the block size limit go away. 

This work would also enable the elimination of the use of buffer heads for the tracking of blocks. Buffer heads are painful for filesystem developers to work with; they also add a lot of overhead for little value. The mapping of pages to blocks is much better managed in the extent-tracking information already found in the filesystem code. About all the page cache would need to worry about is whether any given page is up-to-date with respect to permanent store; the rest can more easily and reliably be managed elsewhere. 

Christoph came back in to point out that he wants the higher-order page cache to reduce application overhead, especially with I/O requests; the filesystem-level solution described by Dave, he said, does not address his problem. Martin Petersen pointed out that smaller I/O granularity may well be forced by host adapters anyway, but Christoph responded that he uses high-end hardware and doesn't have such problems. Dave said, to widespread agreement, that there are two separate problems being described here, and that, in any case, the large filesystem block solution is needed first. 

There were some concerns about interactions between the ELF executable format, which has some alignment assumptions built into it, and a larger filesystem block size. It seems, though, that there are no real problems in this area, especially once one gets away from 32-bit systems. 

The consensus at the end of the session seemed to be that Dave should push forward with his work to move filesystem awareness out of the page cache. It is, he said, a surprisingly small change, but still a significant bit of work. The changes will be opt-in; nobody expects the numerous older filesystems supported by Linux to be updated, and those filesystems need to continue to work. There were concerns about how much of this work will be in the generic filesystem code; there is no desire to see duplicated implementations copied into each filesystem. 

Meanwhile, Christoph was invited to work on supporting larger physically contiguous pages in the page cache. It was made clear, though, that the request for larger allocations would have to be a hint to the allocator; the system cannot depend on those allocations succeeding. In both cases, the next step will be the posting of code for review.
