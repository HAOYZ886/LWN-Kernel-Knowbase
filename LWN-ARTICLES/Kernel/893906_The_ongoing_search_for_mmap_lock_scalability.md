---
title: The ongoing search for mmap_lock scalability
url: https://lwn.net/Articles/893906/
date: "May 6, 2022"
category: "Maple trees; Memory management-mmap sem"
author: "By Jonathan Corbet May 6, 2022 LSFMM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 6, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

There are certain themes that recur regularly at the Linux Storage, Filesystem, Memory-Management, and BPF Summit; among the most reliable is the scalability problems posed by the `mmap_lock` (formerly `mmap_sem`) lock. This topic has come up in (at least) [2013](<https://lwn.net/Articles/548098/>), [2018](<https://lwn.net/Articles/753058/>) ([twice](<https://lwn.net/Articles/753269/>)), and [2019](<https://lwn.net/Articles/787629/>). The [2022 event](<https://events.linuxfoundation.org/lsfmm/>) was no exception, with three consecutive sessions led by Liam Howlett, Michel Lespinasse, and Suren Baghdasaryan dedicated to the topic. There are improvements on the horizon, but the problem is far from solved. 

Lespinasse started with an overview of the problem. The `mmap_lock` is used to serialize changes to a process's address space; in that role, it can cause contention in a number of ways. A multi-threaded process that is generating a lot of page faults, for example, will end up bouncing the lock's cache line around, even though page faults only require a reader lock and can thus be handled in parallel. [![\[Michel
Lespinasse\]](https://static.lwn.net/images/conf/2022/lsfmm/MichelLespinasse-sm.png)](<https://lwn.net/Articles/894082/>) If the threads are performing other types of memory operations, such as mapping or protection changes, they may contend for `mmap_lock` and block each other more directly, even if they are operating on different parts of the address space and should be able to work concurrently. There are also problems when one process accesses another's address space; running `ps` is a simple example. Monitoring tools often run at a relatively low priority; if one acquires the `mmap_lock` then is scheduled out, it can end up blocking the application it is trying to watch. 

Lespinasse had been working on range locking for some time, with the overall goal of splitting the `mmap_lock` into two parts. There would be one lock covering the entire address space that would be held only for short periods, while range locks would be used for longer operations. He couldn't get it working well, though, mostly because it made page faults more expensive. So he has since turned his attention to the longstanding [speculative page-fault handling patches](<https://lwn.net/Articles/730531/>), which should at least help with the cache-line bouncing problem, since they eliminate the need to acquire `mmap_lock` much of the time. 

Liam Howlett then briefly presented the maple tree work; this is a new data structure that was described in detail in [this 2021 article](<https://lwn.net/Articles/845507/>). It is a parallel approach to the `mmap_lock` scalability problem (and more). The maple tree patches successfully replace the current red-black tree for virtual memory areas with about the same performance; in the future, maple trees should more naturally support more scalable access to the process's address space. Meanwhile, the new data structure should be able to take much of the complexity out of the memory-management subsystem. 

Matthew Wilcox joined to note that, in the first phase, the maple tree does not yet bring much in the way of scalability benefits. The `mmap_lock` is still being used in the same places, and the planned ability to use read-copy-update (RCU) with maple trees is not yet there. But it is still a win even at this stage simply due to the reduction in complexity. 

#### The problem with page-fault handling

Wilcox continued by putting up an overview of how the page-fault handling code works to demonstrate where the problems arise: 

```
do_user_addr_fault();
        mmap_read_lock();
        find_vma();
        handle_mm_fault();
          __handle_mm_fault();
            p4d_alloc();
    	pud_alloc();
    	pmd_alloc();
    	handle_pte_fault();
    	  do_fault();
    	    do_read_fault();
    	      do_fault_around();
    	        filemap_map_pages();
    		  /* Stuff under RCU read lock */
        mmap_read_unlock();
```

The point is all of the work done after the `mmap_read_lock()` call, which acquires the `mmap_lock`. The subsequent call to `find_vma()` will find the virtual memory area (VMA) containing the faulting address, and the code requires that VMA to remain stable thereafter — thus the acquisition of the lock. Some of the calls thereafter (the `p?d_alloc()` calls) may perform `GFP_KERNEL` allocations, meaning that they may sleep; that rules out using RCU rather than taking the lock. Once those functions have done their work, though, RCU can be safely used. 

Paul McKenney asked whether perhaps [sleepable RCU (SRCU)](<https://lwn.net/Articles/202847/>) could be used here, especially if a variant could be made that omitted the use of read-side barriers. Wilcox said that SRCU has been tried in the past and shown some performance problems, but things might be [![\[Liam Howlett\]](https://static.lwn.net/images/conf/2022/lsfmm/LiamHowlett-sm.png)](<https://lwn.net/Articles/894083/>) better now. McKenney said those problems are still present, but he might be able to "contort the grace periods" to make things better. 

Wilcox said that, instead, the `p?d_alloc()` functions could gain a GFP-flags argument to tell them not to perform `GFP_KERNEL` allocations; Michal Hocko replied that it might be necessary to dig into a lot of architecture-specific code to make that work. It would have been helpful to have that argument from the beginning, he said, but it will be harder to add now. Johannes Wiener asked whether the flags argument was really needed; perhaps the fault handler could just drop into a slow path if the upper-level page directories are not present. 

The conversation then shifted to the need to hold the lock to prevent changes to the VMA in the first place. The problem, in particular, is that the VMA for a faulting address must continue to contain that address while the fault is being handled; it cannot be allowed to shrink in a way that would leave the address outside. One way to avoid this problem would be to stop changing the VMA in place; instead, it could be replaced using RCU. Lespinasse asked what would happen if there were multiple changes contending for access to the same VMA; it was suggested that a flag indicating that changes are pending could be added. 

The approach taken in the speculative page-fault patches is a little different; rather than using RCU, the code takes a [seqlock](<https://lwn.net/Articles/22818/>). If concurrent changes happen while the fault is being handled, the code would simply start over before committing the previous attempt. This can, Wilcox said, be thought of as spreading out changes in time, rather than spreading them out in space, as is done by by the RCU approach. 

Howlett, meanwhile, concluded that his maple tree work doesn't conflict with either approach to page-fault handling. A remaining problem, though, is the need to preallocate memory — perhaps a fair amount of it — before going into a non-sleepable mode. Much of that memory may ultimately not end up being used, but it must be available; all of this (often useless) allocation and freeing is expensive. It would be nice, Wilcox said, to have a slab call to reserve some number of objects, with the ability to release the unused ones later on. "I'll just get Vlastimil [Babka] to do it", he said. 

Meanwhile, Lespinasse said, this topic has been under discussion for many years; the speculative page fault patches go back to at least 2009. It is about time to start getting some of this code into the mainline. 

Wilcox started to sketch out another variant on the approach. The fault code would, as a first attempt, avoid taking the `mmap_lock` and use RCU for its concurrent updates. If this fails, as it would if memory had to be allocated, for example, then the code would just start over and retry the old-fashioned way. It would not be a huge change, he said, at least for faults on file-backed pages. There was some discussion over whether it made sense to attempt to allocate memory (using `GFP_NOWAIT`) before giving up on the fast path; Wilcox thinks there would be a benefit to making the attempt. 

Hocko asked whether this approach eliminated the need to do range locking, especially once the maple tree is in place too. Howlett answered that range locking adds a lot of complexity, and it may well not be worthwhile. 

Wilcox then suggested that a reader/writer semaphore could be put into the VMA itself; that would have the effect of using the VMA as a sort of range lock. There would still be contention at the VMA level, but it would be an improvement. Developers could then iterate toward a better solution for as long as the community is willing to put up with all the changes. Mel Gorman thought that the contention at the VMA level would be far less severe than with the address-space-wide `mmap_lock`, but Wilcox worried about applications that make terabyte-sized VMAs. Gorman said that applications creating that sort of VMA are almost certainly doing their own memory management anyway. 

The session concluded with the thought that some variant of the above approach might make sense. The proof is always in the code, though. 

#### Speculative page faults for Android

The discussion on speculative page-fault handling was not done, though; Baghdasaryan joined Lespinasse to shift the focus slightly. Baghdasaryan pointed out that Android has a specific interest in speculative page-fault handling because it improves the launch time for applications — something that Android users care a lot about. It is important enough that vendors have been shipping it and have asked for it to be included in the Android common kernel. Multi-threaded applications also benefit from it, he said. 

Why are speculative page faults so helpful in these situations? When Android runs an app, it does so by spawning a new thread which typically starts by mapping a number of VMAs. As the app runs, those VMAs start generating page faults, which create `mmap_lock` contention. Eliminating that contention makes things run much faster. 

The latest speculative page-fault patches were [posted in January](<https://lwn.net/ml/linux-mm/20220128131006.67712-1-michel@lespinasse.org/>), he continued. One of the things holding them back is that few benchmarks show the benefit of this work, and hackbench shows a 4% [![\[Suren
Baghdasaryan\]](https://static.lwn.net/images/conf/2022/lsfmm/SurenBaghdasaryan-sm.png)](<https://lwn.net/Articles/894084/>) regression. That leads to pushback; few people outside of Android see the value of this work. Indeed, many do not know about it at all, so they don't try it out, so there are no reports from use cases that it helps. But everybody can see the added complexity and cost. 

Beyond that, not everybody cares deeply about startup time. Many server applications run for a long time; the time it takes for them to get going is insignificant in comparison. Meanwhile, people who are affected by `mmap_lock` contention have worked around the worst problems long ago. The end result of all this is that the speculative page-fault patches are still being carried in the Android tree, with no clear prospect of going upstream. The Android developers would like to get this work merged, though; the days when Android happily carried a big pile of out-of-tree patches are past. So, he asked, what is the best path forward? 

Lespinasse said that he has seen a lot of people having `mmap_lock` issues; they almost always find some sort of workaround and move on to the next problem, but the issues are still there. The problem is obviously not impossible to deal with, but it is a source of frustration for users. The kernel should not impose this frustration; there are solutions out there, and he wishes that something could go forward. 

Hocko said that getting to a solution will still probably take some time. Different approaches, including speculative page-fault handling and the maple tree, will need to be evaluated; he is not sure that both are required. Lespinasse didn't see a conflict between the two, though; the maple tree is a more efficient red-black tree, he said, but does not give all of the benefits by itself. McKenney suggested that it might make sense to delay the lockless part of the maple-tree patches, giving the speculative page-fault work a chance to show the benefits that it provides. 

The session concluded with no firm outcomes. Chances are, though, that there will be a renewed round of patches showing up soon, and an increased desire to get something upstream. As Baghdasaryan noted, some of this work has been ported forward and backward over many years and been widely tested on a lot of devices. This is not immature work; it should be possible to get at least some of it merged.
