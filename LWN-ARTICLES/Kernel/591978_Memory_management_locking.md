---
title: Memory management locking
url: https://lwn.net/Articles/591978/
date: "March 26, 2014"
category: "Memory management-mmap sem"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Like many parts of the core kernel, the memory management subsystem is highly sensitive to lock contention, which can quickly ruin performance. Davidlohr Bueso has been working on fixing some locking problems in that code; he led a session in the memory management track of the 2014 Linux Storage, Filesystem, and Memory Management Summit to talk about the directions for that work. 

Two locks that show contention problems are the `anon_vma` lock, which controls access to virtual memory areas representing regions of anonymous pages, and the `i_mmap_mutex`, which protects several fields of the `address_space` structure. These locks were once [![](https://static.lwn.net/images/conf/2014/lsfmm/DavidlohrBueso-sm.jpg)](<https://lwn.net/Articles/592089/>) mutexes, but the read-mostly access patterns for those data structures led developers to switch them to reader/writer semaphores (rwsems) instead. The only problem is that performance took a significant hit whenever it became necessary to do a non-trivial amount of writing to those data structures. 

Some of that performance was regained through the application of "rwsem stealing," whereby a thread that is running can grab a lock ahead of another thread which had been waiting for it. But, Davidlohr said, what was missing was the sort of adaptive spinning found in regular mutexes. Even though a mutex is a sleeping lock, a thread trying to acquire it may spin for a while in the hope that the lock will be released soon; doing so can yield a significant performance boost. Adding spinning to the rwsem implementation gets performance back to previous levels for all workloads. So, Davidlohr asked, is there any opposition to merging that code? The response in the room suggested that no such opposition exists. 

In the case of the `anon_vma` lock, there is a strong desire to avoid using a sleeping lock at all. The rwlock mechanism exists for just that use case, but there are fairness issues with rwlocks. Waiman Long has done some work with queued rwlocks, which address those issues while also improving performance. Peter Zijlstra noted that he has rewritten those patches, but is not quite sure what to do with the results. He likes the fairness, but lacks good benchmarks by which to judge them. There are still problems using these locks on virtualized systems. Even so, he is not opposed to merging this code. 

Not everybody feels quite the same way, though. Sagi Grimberg noted that he has code that needs to be able to sleep in functions like `invalidate_page()`, where the `anon_vma` lock can be held. So turning that lock into a non-sleeping lock would clearly create problems. This kind of need comes up in areas like InfiniBand and [RDMA](<http://en.wikipedia.org/wiki/Remote_direct_memory_access>), where work that can potentially sleep has to be done in settings where this lock is held. Mechanisms like [xpmem](<https://code.google.com/p/xpmem/>) also have this problem. 

Rik van Riel suggested that the best way to avoid problems is to get the relevant code upstream as soon as possible, but Davidlohr protested that the performance cost of using a sleeping lock is severe. Peter added that Linus has had "choice words" for authors of code needing a sleeping `anon_vma` lock. So, he said, the right thing to do with the non-sleeping lock patches would be to send them to Linus with an explanation of what would break if they were applied. Then we could all see what Linus chooses to do. 

Davidlohr went on to say that he would also like to restart the discussion of the `mmap_sem` semaphore, which protects many parts of a process's address space. It is often held for too long, he said, creating excessive latencies. We are also serializing too much work. It is not necessary, he said, to lock the entire address space if work is being done on a portion of that space. Perhaps it is time to look at range locking as a way to reduce `mmap_sem` contention? 

Michel Lespinasse responded that, while range locking might make sense, it would be better to work on eliminating long hold times for `mmap_sem`. Rik suggested that, perhaps, it could be turned into a per-virtual-memory-area lock, but Peter responded that this has been tried in the past. The patches have ended up replacing `mmap_sem` contention with contention for a "big VMA lock" instead. 

Jan Kara raised the problem of holding `mmap_sem` when the memory management subsystem calls into filesystem code. Beyond performance problems, this pattern can create lock inversion issues as well. He has been working on eliminating the places where `mmap_sem` is held for filesystem calls for some time; that work is getting closer to being ready. There are just a couple of remaining problem areas, one of which is the page fault handling code, but there are solutions to that problem. 

The other is calls to `get_user_pages()`, which requires that the `mmap_sem` be held by the caller. Jan has been converting callers to `get_user_pages_fast()`, which does not have that requirement. Most of the easy cases have been handled, but a few of the harder issues remain. Sometimes `get_user_pages()` is called in situations where `mmap_sem` has been acquired by higher-level code. The Video4Linux videobuf2 code has some interesting usages of its own which are hard to convert. 

But the most worrisome area is uprobes, which needs to be able to place breakpoints into program code (text) pages when they are brought into memory. This code registers a callback on the creation of virtual memory areas; if need be, it installs breakpoints into text pages when they are instantiated. This results in a call into filesystem code from well within an `mmap()` call. Peter suggested that this one could be fixed by reordering the `mmap()` code. The initial setup work could be done, after which `mmap_sem` would be dropped and the page contents could be filled. That would create a window where a program might be able to access some of the mapped pages before their initialization is complete, but, Peter said, no well-behaved program will access pages created by `mmap()` before that call returns, so there should be no problems. 

The session ended with some inconclusive discussion on rationalizing the naming of the growing family of `get_user_pages()` variants. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
