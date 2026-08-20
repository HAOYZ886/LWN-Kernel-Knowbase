---
title: "Faster page faults with RCU-protected VMA walks"
url: https://lwn.net/Articles/974392/
date: "May 22, 2024"
category: "Memory management-mmap sem"
author: "By Jonathan Corbet May 22, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Looking up a virtual memory area (VMA) in a process's address space, for the handling of page faults or any of a number of other tasks, in multi-threaded processes has long been bedeviled by lock contention in the kernel. As a result, developer gatherings have been subjected to many sessions on how to improve the situation. At the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), developers in the memory-management track met, in a session led by Liam Howlett, to talk about a situation that has improved considerably in recent times, but which still offers opportunities for optimization. 

Howlett began by referring back to [a 2022 LSFMM+BPF session](<https://lwn.net/Articles/893906/>) where Mel Gorman had suggested performing locking during the VMA-walk process at the VMA level itself, rather than locking the whole VMA tree. At that level, Gorman thought, the level of contention would be far less. In current kernels, Howlett said, that is what happens; the fault-handling code will first try locking the VMA tree with the read-copy-update (RCU) read lock, only falling back to the `mmap_lock` if it has to. The VMA of interest can be locked individually once it is located; after the fault is handled, the code calls [`release_fault_lock()`](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/mm.h#L762>), which will either drop the `mmap_lock` or the RCU lock as appropriate. It is not the most elegant solution, he said, but it does hide the details nicely. 

[![\[Liam Howlett\]](https://static.lwn.net/images/conf/2024/lsfmm/LiamHowlett-sm.png)](<https://lwn.net/Articles/974394/>) With regard to performance, he noted that fault-handling actually got slower in the kernels between 5.19 and 6.2 as this work began; distributors were starting to get nervous, he said. But then, in 6.4, the [per-VMA locking](<https://lwn.net/Articles/906852/>) work went in, and performance doubled. By the time 6.6 came around, fault handling was almost three times better than it had been before the work began, a result that he called "pretty awesome". 

For code that needs to walk through the page tables in current kernels, he said, the common pattern is to take the RCU read lock before locating the specific VMA of interest. Code can then call [`lock_vma_under_rcu()`](<https://elixir.bootlin.com/linux/v6.9.1/source/mm/memory.c#L5744>) to try to take the VMA-specific lock and ensure that the VMA does not go away until the work is done. That attempt could fail, though, so code has to be prepared to fall back to `mmap_lock` in that case. Page-fault handling is trickier, though, especially for unpopulated, anonymous memory. In that case, the code may need to examine the neighboring VMAs, and the per-VMA lock won't cover them. Locking multiple VMAs is a quick path to deadlocks, so that is not really an option. The [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) subsystem adds its own special cases as well. 

For anybody else writing code that works through the page tables, he said, looking at the RCU-protected approach rather than taking the contended `mmap_lock` would make sense. There is still a need to work out the best API for all of the use cases out there, though. 

There is also a little problem in that the VMA tree is not atomic in the absence of `mmap_lock`. Holding the per-VMA lock will keep the VMA from going away, but some changes may appear in intermediate states. For example, if an [`munmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call has to split a VMA, the splitting will become visible before the unmapping does. Matthew Wilcox said that developers need to better define what is being promised; if you found the VMA under the RCU lock, the VMA will continue to exist, but it might not still be a part of the process's address space. Suren Baghdasaryan added that some fields of the VMA, including the file pointer, are not stable under RCU. 

The discussion (and the first day) ended with a winding discussion on one of the use cases driving this work: making `/proc/_pid_ /maps` have less impact on the system. There are systems out there with a high-priority process doing work, and a low-priority monitoring process that occasionally needs to read that file. If the low-priority process takes memory-management locks that block the high-priority process, the result is the sort of priority inversion that makes users unhappy. 

Having `/proc/_pid_ /maps` work under the RCU lock prevents that sort of inversion, but at the cost that the VMA tree might change while the file is being read. The contents of that file can always be out of date even in current kernels, since the situation can change immediately after it is read, but now it could also be internally inconsistent. There was some debate over how much of a problem that actually is. There various suggestions of returning sequence numbers that user space could use to detect this situation, or detecting it in the kernel and retrying, perhaps taking the `mmap_lock` after a few failures to ensure that the job gets done. The session came to a close with no definitive conclusions.
