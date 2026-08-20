---
title: Scaling transparent huge pages to 1GB
url: https://lwn.net/Articles/1071716/
date: "May 12, 2026"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet May 12, 2026 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 12, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

As a general rule, when developers talk about huge pages, they are referring to PMD-level pages that are 1MB or 2MB in size, depending on the CPU architecture. Most CPUs can support other huge-page sizes, though. On x86 systems, PUD-level huge pages hold 1GB of data. Providing such large pages transparently to processes has generally not been considered as either feasible or desirable, but Usama Arif is trying to change that assessment. At the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), he led a session in the memory-management track on how to make transparent huge pages (THPs) truly huge. 

On most systems, a 1GB, physically contiguous chunk of memory can be hard to find, especially after the system has run for a while and memory has fragmented. Applications that can make use of such large chunks of memory have also been relatively scarce. It is not surprising that little effort has gone into making a difficult-to-find resource transparently available to processes that are unlikely to benefit from it. But, as Arif began, large-scale installations are now running with terabytes of installed memory. On such systems, a PMD-level huge page is no longer huge. Managing all that memory brings scalability problems; managing it in 1GB chunks can help. 

[![\[Usama Arif\]](https://static.lwn.net/images/conf/2026/lsfmm/UsamaArif-sm.png)](<https://lwn.net/Articles/1071721/>) Applications can gain access to 1GB huge pages now by using the [hugetlbfs](<https://docs.kernel.org/mm/hugetlbfs_reserv.html>) subsystem. But hugetlbfs is a static resource, requiring the establishment of a separate pool at boot time. It provides no fallback if a huge-page allocation request cannot be satisfied. There is a real need, Arif said, for a transparent way to back large applications with 1GB huge pages. He has an RFC patch set to fill this need, [posted in February](<https://lwn.net/ml/all/20260202005451.774496-1-usamaarif642@gmail.com/>), that turned out to be smaller and less invasive than he had expected. 

Arif dove directly into the details of how the management of 1GB transparent huge pages would work. When creating a 2MB PMD-level transparent huge page, current kernels "deposit" an extra (base) page that can later be used to remap the huge page at the PTE level, should the need come to split it. This deposit is made because splitting may be happening in response to memory pressure, so it should be possible to do without having to allocate more memory first. The single page is wasted during the life of the THP, but it serves as a sort of insurance policy for times when memory is scarce. 

In the RFC patch for 1GB THPs, Arif scaled up this policy to match the page size; it deposited pages for the PMD-level page table and 512 PTE-level page tables that would be needed to split the THP. That is about 2MB of wasted memory, which makes for an expensive insurance policy. David Hildenbrand had [questioned](<https://lwn.net/ml/all/ee5bd77f-87ad-4640-a974-304b488e4c64@kernel.org/>) the need for this preallocation — for both PMD-level and PUD-level THPs — so Arif was now considering doing without the page-table deposit. In the session, Hildenbrand said that, if the system is splitting 1GB huge pages, somebody is doing something wrong. Even on the largest systems, those pages are a scarce resource; they should be kept intact if at all possible. 

The question that should be considered, Hildenbrand said, is how to decide which processes should be given 1GB THPs. Kiryl Shutsemau suggested that processes could request them with [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>), but Hildenbrand said that would be problematic for a number of reasons. Shutsemau then wondered about processes asking for 1GB THPs that cannot actually make full use of them; Arif answered that this case is why splitting those pages needs to work. 

Hildenbrand, again, said that splitting those pages should be avoided and advocated for a smarter way of allocating them. Perhaps they could be limited to shared-memory regions, for example. Arif said that would put the burden on user space to set things up properly; he was hoping for a more transparent solution. Lorenzo Stoakes said that 1GB huge pages are a resource that the kernel must maintain control over; Usama said that, by default, they would not be allocated unless the system administrator enabled them. 

Matthew Wilcox said that the right answer was to make 1GB huge pages cheap enough that they are no longer a scarce resource. Johannes Weiner said that there are a lot of applications that could benefit from 1GB THPs, but they (or their users) do not know that. At his employer, they have been rolling out 1GB huge pages extensively, and seeing a lot of performance benefits from them. He suggested handing out 1GB THPs widely, then fixing the cases where they are not fully used. 

Arif moved on to the question of whether supplying 1GB THPs would require using the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>). His patch series works without it, but allocating huge pages of that size can be hard. Part of the problem, he said, is that the memory-management subsystem's compaction code is currently working at the PMD level, so it does not succeed in defragmenting 1GB huge pages. He mentioned some ongoing [work from Rik van Riel](<https://lwn.net/ml/all/20260430202233.111010-1-riel@surriel.com/>) that is aimed at making 1GB chunks easier to allocate. 

Splitting of 1GB THPs is an open question as well, Arif said. The current patch set, when called upon to split such a page, will disassemble it all the way to the PTE level, yielding 262,144 base pages. He is considering only splitting to 512 PMD-level huge pages, then splitting just one of those down to the PTE level. He asked whether that would be an acceptable strategy; relative silence in the room suggested that, at a minimum, there were no real concerns with that idea. 

When he asked whether the `khugepaged` kernel thread should try to assemble 1GB THPs from existing process memory, though, the answer was a clear "no". Allocating them at initial mapping time seems to be the desired approach. Creating them after the fact in response to an `MADV_COLLAPSE` `madvise()` call might be acceptable, though. 

Migration of 1GB THPs is another challenge; finding a 1GB page at the destination node can be difficult. The alternatives are to block migration for these pages, or to split them. Blocking migration is a simple solution, but it loses the transparency aspect, and would break memory hotplug. With splitting, hotplug and NUMA balancing would work, but the 1GB mapping would be lost. Which alternative is best (or least bad) is not clear. 

The group might have tried to debate that problem, but the session was far over its allotted time by this point. Hildenbrand closed it by suggesting that the initial implementation should operate only on shared memory; that would simplify a number of aspects of the implementation. It would also be possible to add a mount option for shmfs that would allow administrators to control access to the feature.
