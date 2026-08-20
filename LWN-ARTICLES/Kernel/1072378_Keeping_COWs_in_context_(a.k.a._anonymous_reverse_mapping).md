---
title: Keeping COWs in context (a.k.a. anonymous reverse mapping)
url: https://lwn.net/Articles/1072378/
date: "May 14, 2026"
category: "Memory management-Object-based reverse mapping; anon vma"
author: "By Jonathan Corbet May 14, 2026 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 14, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

The kernel's reverse-mapping machinery is charged with locating the page-table entries that refer to a given page in memory. The reverse mapping of anonymous pages is handled differently than for file-backed pages. The kernel's implementation of reverse mapping for anonymous pages is, according to Lorenzo Stoakes in his [proposal](<https://lwn.net/ml/all/aec533b2-37a7-4f44-a279-c4aa604206ac@lucifer.local/>) for a memory-management-track session at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), ""a very broken abstraction"", due to its complexity. It also has some performance problems. Stoakes was there to present, in raw form, a proposed replacement that he calls a "COW context". 

The system's page tables map a virtual address to the physical page (if any) the address points to. There is no hardware mechanism, though, to go in the other direction — from a physical page to the page-table entries that refer to it. This reverse mapping is needed by the kernel, though, in order to manage memory. Many years ago, reverse mapping of anonymous pages was done by scanning through all of the page tables in the system; needless to say, that was not the fastest possible solution. Rik van Riel [added a reverse-mapping mechanism](</2002/0124/kernel.php3>) to eliminate the page-table scans in 2002; that machinery was [significantly reworked](<https://lwn.net/Articles/75198/>) by Andrea Arcangeli two years later. The code has evolved considerably in the subsequent two decades, generally in the direction of more complexity. The portion dedicated to the reverse-mapping of anonymous pages is particularly difficult to follow. 

Stoakes began by complaining about that complexity, saying that the code is, in many places, inscrutable. He also lamented the fact that the reverse-mapping code holds locks across an entire fork operation, creating high lock contention and the consequent scalability problems. There are a vast number of kernel objects used to provide reverse mappings, which creates a lot of memory overhead. 

He has been working on an alternative, which can be found (in first-draft form) in [this repository branch](<https://git.kernel.org/pub/scm/linux/kernel/git/ljs/linux.git/log/?h=project/cow-context>). The current reverse-mapping code works at the virtual memory area (VMA) level; since a single process can have large numbers of VMAs, that is a lot of tracking to do. Stoakes is, instead, proposing a "COW context" that is used to track anonymous mappings at the `mm_struct` level. There is one [`mm_struct` structure](<https://elixir.bootlin.com/linux/v7.0.5/source/include/linux/mm_types.h#L1123>) per process; it is the core data structure describing a process's address space. Since only one COW context is needed per process, the system must maintain far fewer of them than if it were a per-VMA structure. 

While the COW context is associated with the `mm_struct`, it is managed as a separate structure. Why is the COW context not directly incorporated into `struct mm_struct`? The COW context structures are linked together to represent the process hierarchy, but they may have to outlive their associated processes. If a process creates a number of children, then exits, its COW context must still exist to maintain the kernel's model of the series of forks that brought about the still-existing processes, since they are likely to continue to share mappings to anonymous pages. 

Each page (strictly, each folio; see [this article](<https://lwn.net/Articles/1064861/>) for a discussion of the difference) gains a pointer to the COW context of the process that first maps it. When the time comes to find all of the mappings of that folio, that pointer can be followed, yielding the COW context at the top of the hierarchy of processes that may have mappings; at that point, it is just a matter of walking the tree of COW contexts to find any other mappings that may exist. There are, of course, complications, many of which tied to the fact that processes can remap their address spaces, meaning that a folio may be mapped at different virtual addresses in different processes. Tracking those remappings adds a fair amount of complexity to the patch set. Stoakes also noted that files mapped with `MAP_PRIVATE` bring pain of their own. 

If a process has created a lot of children, there could be a lot of COW contexts to walk through. To optimize that walk, effort is made to link folios to the COW context at the lowest part of the hierarchy in which they are mapped. As a result, reverse-mapping lookups are quick most of the time. 

One advantage of the new structure is that the lookups can be done under read-copy-update (RCU) protection rather than locking. That is both good and bad; RCU is much faster, but it lacks the synchronization points that locking brings. So he is going to have to introduce some sort of lock with mapping granularity. A ""crazier"" alternative would be to simply tolerate races to an extent; that would require delaying the freeing of page tables until after an RCU grace period, though. 

Stoakes concluded his presentation as the session ran out of time; there was no real discussion resulting from the talk. He acknowledged that the code, as it exists now, has a lot of rough edges and incomplete parts. It is, he said, a research project that might not work out in the end. But, if nothing else, he will have learned a lot more about anonymous reverse mapping.
