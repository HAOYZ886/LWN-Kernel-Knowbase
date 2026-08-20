---
title: Fixing a race in hugetlbfs
url: https://lwn.net/Articles/895540/
date: "May 20, 2022"
category: "Memory management-Huge pages; hugetlbfs"
author: "By Jonathan Corbet May 20, 2022 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 20, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

As the memory-management track at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM) neared its conclusion, Mike Kravetz ran a session remotely to talk about page sharing with hugetlbfs, which is a special filesystem that provides access to huge pages. (See [this article series](<https://lwn.net/Articles/374424/>) for lots of information about hugetlbfs). Hugetlbfs can help to reduce page-table overhead when pages are shared between large numbers of processes, but there is a problem that he is trying to find a solution for. 

One advantage to hugetlbfs, he said, is that processes can share ranges of memory at the PMD page-table level, though the size of the range must be at least 1GB. Sharing huge pages allows the kernel to dispense with the lowest-level page-table pages entirely, saving the memory that would have been used by those pages. This can make a big difference when there is a lot of sharing going on; with a 1GB shared mapping and 10,000 processes sharing it, he said, the result is 39GB of saved memory that would otherwise be used for page-table pages. Hugetlbfs, when used this way, is solving the same problem targeted by the [`mshare()` proposal](<https://lwn.net/Articles/895217/>), but the mechanism is different; rather than sharing page-table pages, hugetlbfs just eliminates the need for those page-table pages. 

That is nice, but there is a problem lurking therein. When a process's mapping to a hugetlbfs page is deleted, a call to [`huge_pmd_unshare()`](<https://elixir.bootlin.com/linux/v5.17.8/source/mm/hugetlb.c#L6532>) results. This can also happen when changing a mapped page's attributes with `mprotect()`. If a fault happens on a page while this unsharing is happening, though, the result is an "ugly race" that can create invalid page-table entries. The problem is easy to provoke from user space, he said. 

This problem was fixed by [commit c0d0381ade79](<https://git.kernel.org/linus/c0d0381ade79>) in 2020, which uses the `i_mmap_rwsem` semaphore to synchronize the unshare operation. It must also be held during page-fault processing, of course, to prevent the race from happening. This fix created a new problem, though, because `i_mmap_rwsem` is held for the duration of a number of potentially long-running operations, including truncation and hole punching. That can cause long delays, with latencies greater than two seconds in his testing. 

To address this problem, he said, the previous fix should be reverted. Instead, a per-VMA reader/writer semaphore should be used to synchronize these operations. That limits the contention and makes the worst case a lot better. 

He asked the assembled developers what they thought of this fix, and was greeted with resounding silence. After some time, Matthew Wilcox observed that Kravetz had "broken people's brains" with the presentation. Kravetz replied that he would post another RFC patch soon and the conversation could continue from there.
