---
title: The LRU lock and mmap_sem
url: https://lwn.net/Articles/753058/
date: "April 30, 2018"
category: "Memory management-mmap sem"
author: "By Jonathan Corbet April 30, 2018 LSFMM"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 30, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The kernel's memory-management subsystem has to manage a great deal of concurrency; that leads to an ongoing series of locking challenges that sometimes seem intractable. Two recurring locking issues — the LRU locks and the `mmap_sem` lock — were the topic of sessions held during the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit. In both cases, it quickly became clear that, while some interesting ideas are being pursued, easy solutions are not on offer. 

#### Too-frequently used LRU locks

The kernel maintains a set of least-recently-used (LRU) lists to track both anonymous and page-cache pages; when the time comes to reclaim some memory for other uses, the pages that have been idle the longest are the first to go, since they are, with luck, the pages that are least likely to be needed in the near future. The LRU lists (which exist for each NUMA node) are dynamic, with pages being added, removed, or reordered frequently. Daniel Jordan started his session by noting that Oracle has been running into problems with contention for the LRU locks that serialize access to the lists; about 1% of query time, he said, is spent waiting on those locks. 

The problem, he said, is that the LRU lock is "a big hammer" controlling access to an entire LRU list. There should be no need for such a hammer; [![\[Daniel Jordan\]](https://static.lwn.net/images/conf/2018/lsfmm/DanielJordan-sm.jpg)](<https://lwn.net/Articles/753060/>) multiple threads should be able to operate on different parts of an LRU list concurrently. Getting there, he said, requires moving to a special type of per-page lock that uses the list structure itself. 

Under his proposed scheme, the first step to remove a page from an LRU list is to put a special value into the "next" pointer of the _previous_ page on the list. A compare-and-swap (CAS) operation would be used to change this pointer, making it possible to detect contention with another thread trying to make a change at the same place at the same time. The page being removed would also have its "next" pointer changed to a sentinel value as well. 

At this point, any other thread traversing the LRU list will, when it hits the sentinel value, know that things are being changed; it will then spin on the pointer until it returns to a normal value. With traversals and concurrent changes blocked, the page of interest can now be removed from the LRU. The "next" pointer in the previous page, which remains on the list, can now be set to the page that followed the removed page, removing the lock and re-enabling concurrent operations. 

A similar algorithm can be used for adding pages to the list. The list head itself can also be set to the sentinel value when pages are added to the front of the list. 

Jordan said that contention is almost never encountered when using this algorithm, so the problems with the LRU lock essentially go away. Johannes Weiner suggested that it could be reduced further during addition operations by searching for an uncontended point rather than spinning; the exact position of new pages in the list isn't particularly important. Andrew Morton said that this algorithm could prove to be useful for a number of busy lists in the kernel. 

Dave Hansen, instead, said that this idea was "cool", but that the real contention problem is the zone lock, which should be dealt with first. He noted that Aaron Lu has [done some work](<https://lwn.net/ml/linux-kernel/20180320085452.24641-1-aaron.lu@intel.com/>) in this area, and suggested that this "CAS trick" could perhaps work there as well. Hugh Dickins said that the algorithm is more interesting as a general approach to list manipulation than as a specific solution to LRU-lock contention, which isn't a problem for everybody. The session then wound down with a brief discussion of perhaps increasing batching in page management as another way of reducing contention. 

#### mmap_sem

The `mmap_sem` semaphore is used to control access to a process's address space — and to a variety of other, related data structures. It has long been a contention point in the memory-management subsystem, but it has proved resistant to change. Laurent Dufour's session, held immediately after the LRU-lock discussion, started with a complaint that `mmap_sem` is the source of a great deal of contention on large systems that are running a lot of threaded applications. Can something be done about that? 

The place to start, he said, is figuring out just what `mmap_sem` protects. That is not an easy answer to find. It covers access to many fields in the [`mm_struct` structure](<https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L349>). It is also used for the [virtual memory area (VMA)](<https://elixir.bootlin.com/linux/latest/source/include/linux/mm_types.h#L268>) red-black tree, the process VMA list, and various fields within the VMA structure itself. But that is just a beginning, he said; a serious audit will be needed to find the rest. 

What are the options for reducing `mmap_sem` contention? One is [speculative page-fault handling](<https://lwn.net/Articles/730531/>), an area Dufour has been working on for a while. It allows the handling of page faults, in many cases, without the need to grab `mmap_sem` at all. [![\[Laerant Dufour\]](https://static.lwn.net/images/conf/2018/lsfmm/LaurentDufour-sm.jpg)](<https://lwn.net/Articles/753061/>) Breaking up `mmap_sem` into finer-grained locks is possibly interesting he said. A variant of that is [range locking](<https://lwn.net/Articles/724502/>), which would support locking a portion of the address space rather than the whole thing; range locking won't solve all of the problems, though. There may be places where [SRCU](<https://lwn.net/Articles/202847/>) could be used to reduce contention. Finally, he noted that splitting and merging of VMAs is a contention point that could perhaps be resolved by deferring the merging of VMAs. 

Hansen said that there were a lot of solutions in this list, but asked for a list of the problems being solved. He noted that even read access to `mmap_sem` hurts, since it bounces the reader count between processor caches. He suggested picking one specific problem and working on a solution. 

Michal Hocko said that the real problem is applications "pretending they can have thousands of threads and it will still work". He, too, suggested prioritizing problems and picking the one that seems most important. Speculative page faults are nice, he said, but the patch also adds a lot of complexity to the page-fault path — which is already complex. There has been a lack of use cases that would show a real benefit from range locking, so that work has been stalled for a while. Smaller steps, he said, would be a better way to go. 

There was a side discussion on range locking and how to pick the proper range to lock. It was pointed out that, for many operations, locking the range covered by a specific VMA would not be enough; it would also be necessary to lock one page on either side of the VMA to prevent concurrent merging. It is not clear that range locking is worth the effort, especially since many applications consist of one large VMA containing the bulk of the address space. 

One area of contention is the `mprotect()` system call, which can cause a lot of splitting and merging of VMAs. Applications could reduce that contention in many cases by using [memory protection keys](<https://lwn.net/Articles/689395/>) instead. It was also suggested that the kernel could just avoid merging VMAs after `mprotect()` calls. Memory-management developers have long wanted to minimize the number of VMAs, but perhaps that doesn't really matter. 

Hocko brought the session to a close with the conclusion that there is a lot of work to do in this area, and that no "single bullet" exists to solve the problem. He suggested getting rid of the worst abuses of `mmap_sem` as a starting point; there are `/proc` interfaces that use it, for example. Once that has been done, maybe the search could begin for a sane range-locking approach. Hansen said that the lock itself is often not the problem, and that the place to start is by better documenting how `mmap_sem` is actually used.
