---
title: "Ways to reclaim unused page-table pages"
url: https://lwn.net/Articles/893726/
date: "May 9, 2022"
category: Memory management
author: "By Jonathan Corbet May 9, 2022 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 9, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

One of the memory-management subsystem's most important jobs is reclaiming unused (or little-used) memory so that it can be put to better use. When it comes to one of the core memory-management data structures — page tables — though, this subsystem often falls down on the job. At the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), David Hildenbrand led a session on the problems posed by the lack of page-table reclaim and explored options for improving the situation. 

Page tables, of course, contain the mapping between virtual and physical addresses; every virtual address used by a process must be translated by the hardware using a page-table entry. These tables are arranged in a hierarchy, three to five levels deep, and can take up a fair amount of memory in their own right. On an x86 system, a 2TB mapping will require 4GB of bottom-level page tables, 8MB of the next-level (PMD) table, and smaller amounts for the higher-level tables. These tables are not swappable or movable, and sometimes get in the way of operations like hot-unplugging memory. 

Page tables are reclaimed by the memory-management subsystem in some cases now. For example, removing a range of address space with [`munmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) will clean up the associated page tables as well. But there are situations that create empty page tables that are not reclaimed. Memory allocators tend to map large ranges, for example, and to only use parts of the range at any given time. When a given sub-range is no longer needed, the [![\[David
Hildenbrand\]](https://static.lwn.net/images/conf/2022/lsfmm/DavidHildenbrand-sm.png)](<https://lwn.net/Articles/893758/>) allocator will use [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) to release the pages, but the associated page tables will not be freed. Large mapped files can also lead to empty page tables. These are all legitimate use cases, and they can be problematic enough; it is also easy to write a malicious program that fills memory with empty page tables and brings the system to a halt, which is rather less legitimate. 

In all of these cases, many of the resulting page tables are empty and of no real use; they could be reclaimed without harm. One step in that direction, Hildenbrand said, is [this patch set from Qi Zheng](<https://lwn.net/ml/linux-kernel/20220429133552.33768-1-zhengqi.arch@bytedance.com/>). It adds a reference count (`pte_ref`) to each page table page; when a given page's reference count drops to zero, that page can be reclaimed. 

Another source of empty page-table pages is mappings to the shared zero page. This is a page maintained by the kernel, containing only zeroes, which is used to initialize anonymous-memory mappings. The mapping is copy-on-write, so user space will get a new page should it ever write to a page mapped in this way. It is easy for user space to create large numbers of page-table entries pointing to the zero page, once again filling memory with unreclaimable allocations. This can be done, for example, by simply reading one byte out of every 2MB of address space mapped as anonymous memory. These page-table pages could also be easily reclaimed, though, if the kernel were able to detect them; there is no difference (as far as user space can tell) between a page-table page filled with zero-page mappings and a missing page-table page. At worst, the page-table page would have to be recreated should an address within it be referenced again from user space. 

There are other tricks user space can use to create large numbers of page-table pages. Some of them, like so many good exploits, involve the [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) system call. 

So what is to be done? Reclaiming empty page-table pages is "low-hanging fruit", and the patches to do so already exist. The kernel could also delete mappings to the zero page while scanning memory; that would eventually empty out page-table pages containing only zero-page mappings, and allow those to be reclaimed as well. Ordinary reclaim unmaps file-backed pages, which can create empty page-table pages, but that is not an explicit objective now. It might be possible to get reclaim to focus on pages that are close to each other in the hopes of emptying out page-table pages, which could then be reclaimed as well. 

There are some other questions to be answered as well, though. Some page-table pages may be actively used, even if they just map the zero page; reclaiming them could hurt performance if they just have to be quickly recreated again. There are also questions about whether it makes sense to reclaim higher-level page tables. Reclaiming empty PMD pages might be worthwhile, Hildenbrand said, but there is unlikely to be value in trying to go higher. 

When should this reclaim happen? With regard to empty page-table pages, the answer is easy — as soon as they become empty. For zero-page mappings, the problem is a little harder. They can be found by scanning, but that scanning has its own cost. So perhaps this scanning should only happen when the kernel determines that there is suspicious zero-page activity happening. But defining "suspicious" could be tricky. Virtual machines tend to create a lot of zero-page mappings naturally, for example, so that is not suspicious on its own; it may be necessary to rely on more subjective heuristics. One such might be to detect when a process has a high ratio of page-table pages to its overall resident-set size. 

Even when a removable page-table page has been identified, he said, removing it may not be trivial. There is a strong desire to avoid acquiring the `mmap_lock`, which could create contention issues; that limits when these removals can be done. Removing higher-level page-table pages is harder, since ordinary scanning can hold a reference to them for a long time. Waiting for the reference to go away could stall the reclaim process indefinitely. 

With regard to what can be done to defeat a malicious user-space process, Hildenbrand said that the allocation of page-table pages must be throttled somehow. The best way might be to enforce reclaim of page-table pages against processes that are behaving in a suspicious way. One way to find such processes could be to just set a threshold on the amount of memory used for page-table pages, but that could perhaps be circumvented simply by spawning a lot of processes. 

At this point, the session wound down. Yang Shi suggested at the end that there might be some help to be found in the [multi-generational LRU](<https://lwn.net/Articles/856931/>) work, if and when that is merged. It has to scan page tables anyway, so adding the detection of empty page-table pages might fit in relatively easily.
