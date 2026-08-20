---
title: "Concurrent page-fault handling with per-VMA locks"
url: https://lwn.net/Articles/906852/
date: "September 5, 2022"
category: "Memory management-mmap sem; Releases-6.4"
author: "By Jonathan Corbet September 5, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 5, 2022

The kernel is, in many ways, a marvel of scalability, but there is a longstanding pain point in the memory-management subsystem that has resisted all attempts at elimination: the `mmap_lock`. This lock was [inevitably a topic](<https://lwn.net/Articles/893906/>) at the [2022 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmm2022/>) (LSFMM), where the idea of using per-VMA locks was raised. Suren Baghdasaryan has [posted an implementation of that idea](<https://lwn.net/ml/linux-kernel/20220901173516.702122-1-surenb@google.com/>) — but with an interesting twist on how those locks are implemented. 

The `mmap_lock` (formerly called `mmap_sem`) is a reader/writer lock that controls access to a process's address space; before making changes there (mapping in a new range, for example), the kernel must acquire that lock. Page-fault handling must also acquire `mmap_lock` (in reader mode) to ensure that the address space doesn't change in surprising ways while a fault is being resolved. A process can have a large address space and many threads running (and incurring page faults) concurrently, turning `mmap_lock` into a significant bottleneck. Even if the lock itself is not contended, the constant cache-line bouncing hurts performance. 

Many attempts at solving the `mmap_lock` scalability problem have taken the form of [speculative page-fault handling](<https://lwn.net/Articles/730531/>), where the work to resolve a fault is done without taking `mmap_lock` in the hope that the address space doesn't change in the meantime. Should concurrent access occur, the speculative page-fault code drops the work it has done and retries after taking `mmap_lock`. Various implementations have been shown over the years and they have demonstrated performance benefits, but the solutions are complex and none have managed to convince enough developers to be merged into the mainline kernel. 

An alternative approach that has often been considered is range locking. Rather than locking the entire address space to make a change to a small part of it, range locking ensures exclusive access to the address range of interest while allowing accesses to other parts of the address space to proceed concurrently. Range locking turns out to be tricky as well, though, and no implementation has gotten close to being considered for merging. 

#### VMA locking

A process's address space is described by a sequence of virtual memory areas (VMAs), represented by [`struct vm_area_struct`](<https://elixir.bootlin.com/linux/v5.19.6/source/include/linux/mm_types.h#L392>). Each VMA corresponds to an independent range of address space; an [`mmap()`](<https://man7.org/linux/man-pages/man2/munmap.2.html>) call will normally create a new one, for example. Consecutive VMAs with the same characteristics can be merged; VMAs can also be split if, for example, a process changes the memory protections on a portion of the range. The number of VMAs varies from one process to the next, but it can grow to be quite large; the Emacs process within which this article is being written has over 1,100 of them, while `gnome-shell` has over 3,100. 

At LSFMM this year, Matthew Wilcox suggested that the range-locking problem could be simplified by turning it into a VMA-locking problem. Since each VMA covers a range of the address space, locking the VMA would be equivalent to locking that range. The result would have much coarser resolution than true range locking, but it might still be good enough to be worth the effort. 

Baghdasaryan's patch set is the attempt to find out if that is the case. But, of course, it immediately ran into the complexities of memory-management subsystem locking. There are two distinct types of locks that need to be taken on a VMA: 

  * Page-fault handling needs to ensure that the VMA remains present while a fault is being resolved and that it doesn't change in problematic ways. This work can be done concurrently with the handling of other faults or a number of other tasks, though. So the page-fault handler needs to take what is essentially a read lock. 
  * Address-space changes will need exclusive access to one or more VMAs; while (for example) a VMA is being split, no other part of the kernel can be allowed to do anything with any of the parts. So these types of changes require a write lock. 

The original idea had been to use a reader/writer lock for this task, but that led to another problem: write locks often need to be applied to multiple VMAs at once. It would be possible to implement this with reader/writer locks but, as Baghdasaryan pointed out in the cover letter: ""Tracking all the locked VMAs, avoiding recursive locks and other complications would make the code more complex"". There is surprisingly little desire for more complexity in the core memory-management code, so he went in search of a different solution. 

#### The implementation

The scheme that emerged was a combination of a reader/writer lock and a sequence number that is added to every VMA, but also to the [`mm_struct` structure](<https://elixir.bootlin.com/linux/v5.19.6/source/include/linux/mm_types.h#L481>) that describes the address space as a whole. If the sequence number in a given VMA is equal to the `mm_struct` sequence number, then that VMA is considered locked for modification and inaccessible for concurrent page-fault handling. If the two numbers disagree, no lock exists and concurrent access is possible. 

When a page fault occurs, the handler will first attempt to read-lock the per-VMA lock; if that fails then it falls back to acquiring the full `mmap_lock` as is done now. If the read lock succeeds, though, the handler must also check the sequence numbers; if the sequence number for the relevant VMA matches that in the `mm_struct` (which cannot change as long as `mmap_lock` is held), then other changes are afoot and handling must, once again, fall back to taking `mmap_lock`. Otherwise the VMA is available and the fault can be handled without locking the address space as a whole. The read lock will be released once that task is complete. 

When the memory-management system must make address-space changes, instead, it must lock each of the VMAs that will be affected. The first step is to take a write lock on `mmap_lock`, then, for each VMA, it will acquire the reader/writer lock in write mode (potentially waiting for any existing readers to let go of it). That lock is only held for long enough to set the VMA's sequence number equal to the `mm_struct` sequence number, though. Once that change has been made, the VMA is locked even after the reader/writer lock is released. 

Another way to describe this is to say that the per-VMA reader/writer lock really only exists to protect access to the per-VMA sequence number, which is the real per-VMA lock. 

After the kernel has locked all of the relevant VMAs, whatever changes need to be made can proceed. It will not be possible to handle page faults within those VMAs during this time (as is the case now), but other parts of the address space will be unaffected. Once the work is complete, all of those VMAs can be unlocked by simply increasing the `mm_struct` sequence number. There is no need to go back to each locked VMA — or even to remember which ones they are. 

There are, of course, plenty of other details that have been glossed over here, including the need to bring VMAs under read-copy-update protection so that they can be looked up without holding `mmap_lock`. But the locking scheme is the core that makes it all work. According to Baghdasaryan, the resulting performance increase is about 75% of that achieved with the speculative page-fault patches, so it's still leaving some performance on the table. But, he said: ""Still, with lower complexity this approach might be more desirable"". 

This work is deemed to be a proof-of-concept at this point. Among other things, it only handles faults on anonymous pages and, even then, only those that are not in swap. Support for swapped and file-backed pages can be added later, he said, if the approach seems worth pursuing. Answering that question may take a while; core memory-management patches tend not to be merged quickly, and this discussion is just beginning. But if it works out, this patch set could be a step in the direction of the long-wished-for range-locking mechanism for process address spaces.
