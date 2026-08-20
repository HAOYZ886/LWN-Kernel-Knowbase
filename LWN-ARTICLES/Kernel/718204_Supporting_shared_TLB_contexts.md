---
title: Supporting shared TLB contexts
url: https://lwn.net/Articles/718204/
date: "March 28, 2017"
category: "Memory management-Translation lookaside buffer"
author: "By Jonathan Corbet March 28, 2017 LSFMM 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 28, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017>)

A processor's translation lookaside buffer (TLB) caches the mappings from virtual to physical addresses. Looking up virtual addresses is expensive, so good performance often depends on making the best use of the TLB. In the memory-management track of the 2017 Linux Storage, Filesystem, and Memory-Management Summit, Mike Kravetz described a SPARC processor feature that can improve TLB performance and explored ways in which that feature could be supported. 

On most processors, context switches between processes are expensive operation because they force the contents of the TLB to be flushed. SPARC differs, though, in that TLB entries carry a tag associating them with a specific context. Since the processor knows to ignore TLB entries that do not correspond to the process that is executing, there is no need to flush the TLB on context switches. That takes away much of the context-switch penalty, and, as a result, improves performance. 

The SPARC context register has been supported in Linux for a long time. But, Kravetz said, recent SPARC processors have added a second register, meaning that any given process can be associated with two independent contexts at the same time. Kravetz, an Oracle employee, said that this [![\[Mike Kravetz\]](https://static.lwn.net/images/conf/2017/lsfmm/MikeKravetz-sm.jpg)](<https://lwn.net/Articles/718209/>) helps these processors support "the most important application in the world" — the Oracle database — which is built around a set of processes working on a large shared-memory area. If the second context ID is assigned to that area, then the associated TLB entries can be shared across all of those processes. 

He has posted [a patch set](<https://lkml.org/lkml/2016/12/16/508>) allowing this register to be used for shared-memory areas. The patch is "80% SPARC code", though, so nobody but Dave Miller (the SPARC maintainer) has looked at it, he said. His hope was to draw more attention to this feature and work out the best way to expose the functionality of this second context ID to user space. 

His thinking is to have a special virtual memory area (VMA) flag to indicate a memory region with a shared context. But that leaves the question of how that flag should be set; Kirill Shutemov observed that it could be difficult to provide a sane interface for this feature. Kravetz's proposal added a special flag to the `mmap()` and `shmat()` system calls. One nice feature of this approach is that it does not require exposing the shared-context ID to user space. Instead, the kernel sees that the flag was set, assigns a context ID, and ensures that all processes mapping the same area at the same virtual address use the same context. 

Matthew Wilcox suggested that perhaps `madvise()` would be a better system call for this functionality. The problem with `madvise()`, Kravetz said, is that it creates an inherent race condition. The shared context ID is stored in the page-table entries, so it needs to be set up before any of those entries are created. In particular, it needs to be in place before the process faults any of the pages in the shared region. Otherwise, those prematurely faulted pages will not be associated with the shared ID. 

Kravetz's first patch set only supported pages mapped from [hugetlbfs](<https://lwn.net/Articles/374424/>), which was enough to cover the Oracle shared-memory area. But he noted that it would be nice to cover executable mappings as well. While that would enable the shared ID to be used with shared libraries; the more immediate use case was the Oracle database executable, of course. Dave Hansen reacted to this idea by observing that Oracle seems to be trying to glue its multiprocess implementation back into a single process. (This feature, it should be noted, would not play well with address-space layout randomization, since all mappings must be to the same virtual address). 

It was suggested that, in essence, hugetlbfs is a second memory-management subsystem for the kernel, providing semantics that the original lacked. DAX, perhaps, is developing into a third. The shared-context flag is needed because hugetlbfs is a second subystem; otherwise, things would be shared more transparently. So perhaps the real answer is to get rid of hugetlbfs? The problem with that idea, Andrea Arcangeli said, is that hugetlbfs will always have a performance advantage over transparent huge pages because the huge pages are reserved ahead of time. There are not many hugetlbfs users out there, but those few really want it. 

Arcangeli went on to say that the real problem with TLB performance is that Linux is still using small (4KB) pages; someday that page size is going to have to increase. Shutemov said that increase would be an ABI break, but Arcangeli countered that, when the x86-64 port was done, care was taken to not expose any boundaries smaller than 2MB to user space. That takes care of most potential ABI issues (on that architecture), but there are still cases where user space sees the smaller page size — `mprotect()` calls, for example. So Linux will not be able to get completely away from small pages anytime soon. 

As the end of the session approached, Rik van Riel pulled the conversation back to the main topic by asking if there were any action items. It seems that there are no known bugs in Kravetz's patch set, other than the fact that it is limited to hugetlbfs, which ignores memory-allocation policies, cpusets, and more. Mel Gorman said that, since hugetlbfs is its own memory-management subsystem, it can do what it wants in that area; Michal Hocko suggested simply documenting the things that don't work properly. The final question came from Hansen, who asked whether this feature was really important or not. The answer seems to be "yes, because Oracle wants it".
