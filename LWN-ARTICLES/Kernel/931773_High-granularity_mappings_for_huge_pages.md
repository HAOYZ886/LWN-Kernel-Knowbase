---
title: "High-granularity mappings for huge pages"
url: https://lwn.net/Articles/931773/
date: "May 17, 2023"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet May 17, 2023 LSFMM+BPF"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 17, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The use of huge pages can make memory management more efficient in a number of ways, but it can also impose costs in the form of internal fragmentation and I/O amplification. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), James Houghton ran a session on a scheme to get the best of both worlds: using huge pages while maintaining base-page mappings within them. 

The problem to be solved, Houghton began, is that performance requirements argue for the use of huge pages to back virtual machines. So far, so good; the hugetlbfs system can be used to implement such a policy. But providers also want to be able to migrate virtual machines using [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>). Typical migration schemes will pre-copy the machine to be moved, then use `userfaultfd()` to catch any post-copy writes and re-copy the changed pages before committing the moved machine to the new host. But, if huge pages are in use, the only information available is that an entire (huge) page has been touched. It would be nice to just copy the 4KB base page that the process changed, but there is no way to know where within a 1GB huge page that change was made. 

[![\[James Houghton\]](https://static.lwn.net/images/conf/2023/lsfmm/JamesHoughton-sm.png)](<https://lwn.net/Articles/931776/>) To address this problem, he posted [a patch set](<https://lwn.net/ml/linux-mm/20230218002819.1486479-1-jthoughton@google.com/>) in February implementing high-granularity mapping for 1GB hugetlbfs pages. This series allows huge pages to be mapped at the PTE (base-page) level without splitting them. For now, this work only supports x86 systems, but there is an arm64 version written that has not yet been posted to the lists. 

There are some challenges in implementing a mechanism like this, he said, starting with the user-space ABI to control it. The approach taken was to [implement an `MADV_SPLIT` operation](<https://lwn.net/ml/linux-mm/20230218002819.1486479-10-jthoughton@google.com/>) for [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) — an approach that David Hildenbrand described as "confusing". This ABI seems likely to change in the future if this work proceeds. There are some implementation challenges as well, Houghton said; keeping track of the page-table entries for high-granularity-mapped pages requires access to the page middle directory (PMD) level of the page-table hierarchy, but the internal APIs do not always pass that in. A number of other internal changes are needed to implement this feature as well, making the implementation of high-granularity mappings into a 46-part patch set. 

Michal Hocko suggested that, perhaps, the real problem is that this use case should not be using hugetlbfs as its backing store. Instead, a different way of accessing huge pages that does not bring hugetlbfs's baggage should be found, he said. If the desire is to keep the huge pages from being reclaimed, [`mlock()`](<https://man7.org/linux/man-pages/man2/mlock.2.html>) can be used. Houghton answered that hugetlbfs can guarantee that huge pages will be available, which is a necessary feature; trying to solve the problem without hugetlbfs would also require copying many of its quirks into the core memory-management code. Hocko said that, perhaps, the right approach would be to create a driver to obtain huge pages from the [CMA](<https://lwn.net/Articles/486301/>) pool; Houghton thought that might work. 

Another audience member asked whether the transparent hugepage mechanism, which already supports high-granularity mappings, could be enhanced to provide guaranteed allocations for this use case. Another attendee said that the memory-management subsystem should, eventually at least, be prepared to handle folios of any size, so the need for special-purpose code for huge pages should go away. If this work could get the kernel closer to that point, that would be a good thing. 

Houghton felt, though, that implementing the needed functionality in the core memory-management code would be a lot of work, while containing it in a separate implementation simplifies the task considerably. That implementation shows the core changes that would eventually be needed, and is thus a step in the right direction. It is not clear that the rest of the room was in agreement with this position, though; this feature may not be headed for the mainline in the near future.
