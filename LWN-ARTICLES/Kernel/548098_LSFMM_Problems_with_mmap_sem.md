---
title: "LSFMM: Problems with mmap_sem"
url: https://lwn.net/Articles/548098/
date: "April 23, 2013"
category: "Memory management-mmap sem"
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

The top-level data structure describing a process's virtual address space is called `struct mm_struct`; it contains a reader/writer semaphore called `mmap_sem` that serializes changes to the structure and to a number of related data structures. The opening session in the memory-management track at the 2013 Linux Filesystem, Storage, and Memory Management Summit looked at a specific problem relating to this semaphore and how it might be addressed. 

The problem has to do with the order in which locks are acquired. For normal filesystem operations, the filesystem code will obtain any locks it requires; the memory management subsystem will then grab `mmap_sem` should that be required — to bring a read or write buffer into RAM, for example. When a page fault happens, though, the lock ordering is reversed: first `mmap_sem` is taken, then the filesystem code is invoked to bring the needed page into RAM. Inconsistent lock ordering is a highly effective way to deadlock the system, so this behavior is a bit of a problem. It has, for the most part, been worked around in the filesystem code, but those workarounds are getting harder to maintain. 

As an example, one of the more problematic locks is `i_mutex`, which protects access to an `inode` structure. In this case, lock inversion problems have been worked around by using a different lock in the page fault path; this approach works but lacks elegance. There has been talk of turning `i_mutex` into a "range lock," so that only a portion of a file would need to be locked at any given time; that would enable filesystems to avoid lock ordering problems most of the time. But the real point that was made by filesystem developers is that `mmap_sem` is entirely unneeded within the filesystem code; perhaps the memory management code could just drop that lock before calling into filesystems? 

The problem with that idea, according to Michel Lespinasse, is that filesystem calls result in the placement of pages into the page cache, and, from there, into the relevant process's page tables. Changes to page tables cannot be done without holding `mmap_sem`. It was suggested that the problem could be solved by dropping `mmap_sem` before calling into the filesystem, then scanning the page cache to update page table entries after the filesystem code has done its work. This approach was seen as workable, but, it was pointed out, if `mmap_sem` is to be dropped it must be dropped in _every_ path that calls into the filesystem code. 

One might think that there shouldn't be too many such paths, but the situation is a bit more complicated that that, thanks to `get_user_pages()`, which is used by kernel code to fault in user-space pages. Jan Kara reported that he has audited some 50 `get_user_pages()` call sites in the kernel and found that most of them can use `get_user_pages_fast()` instead. That's an important change; `get_user_pages()` requires that `mmap_sem` be held by the caller, while `get_user_pages_fast()` does not. So changing all those call sites would eliminate a long list of paths by which filesystem code can be called with `mmap_sem` held. 

Another approach might be to use a sequence counter to track changes to the address space; that would eliminate the need for a real lock much of the time. But that approach adds other challenges: measures would have to be taken to keep the relevant virtual memory area (VMA) and file structures around while the filesystem code is doing its thing, for example. Changes to the VMA itself are also possible and would have to be watched for. There is also a possibility that some architectures would have to be changed: if the counter indicates that a change has been made, it may be necessary to retry an attempt to satisfy a page fault, but not all architectures have fault retry code in place. 

A possible worst case scenario would be a process that incurs a page fault on a given VMA. While the fault is being handled, the memory could be unmapped, causing the VMA to be deleted; then another, unrelated VMA could be instantiated for the same address range. There is little good that could come from this kind of confusion. A sequence counter that incremented for every address space change would catch such occurrences, but Hugh Dickins worried that it could often be incremented for unrelated events, slowing page fault handling. 

The session had few in the way of definitive conclusions. Jan said that he would continue to look at users of `get_user_pages()` with the goal of eventually eliminating the need to hold `mmap_sem`. That will take some work since, he said, some device drivers include complicated `fault()` handlers that do tricky things. The `fault()` function could, perhaps, be called without `mmap_sem` someday, but that may come at the cost of adding a separate `fault_locked()` for the few places where `mmap_sem` is truly needed.
