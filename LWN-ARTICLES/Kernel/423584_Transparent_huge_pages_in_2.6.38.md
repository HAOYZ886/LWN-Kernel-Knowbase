---
title: Transparent huge pages in 2.6.38
url: https://lwn.net/Articles/423584/
date: "January 19, 2011"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet January 19, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 19, 2011

The memory management unit in almost any contemporary processor can handle multiple page sizes, but the Linux kernel almost always restricts itself to just the smallest of those sizes - 4096 bytes on most architectures. Pages which are larger than that minimum - collectively called "huge pages" - can offer better performance for some workloads, but that performance benefit has gone mostly unexploited on Linux. That may change in 2.6.38, though, with the merging of the transparent huge page feature. 

Huge pages can improve performance through reduced page faults (a single fault brings in a large chunk of memory at once) and by reducing the cost of virtual to physical address translation (fewer levels of page tables must be traversed to get to the physical address). But the real advantage comes from avoiding translations altogether. If the processor must translate a virtual address, it must go through as many as four levels of page tables, each of which has a good chance of being cache-cold, and, thus, slow. For this reason, processors maintain a "translation lookaside buffer" (TLB) to cache the results of translations. The TLB is often quite small; running `cpuid` on your editor's aging desktop machine yields: 

```
cache and TLB information (2):
          0xb1: instruction TLB: 2M/4M, 4-way, 4/8 entries
          0xb0: instruction TLB: 4K, 4-way, 128 entries
          0x05: data TLB: 4M pages, 4-way, 32 entries
```

So there is room for 128 instruction translations, and 32 data translations. Such a small cache is easily overrun, forcing the CPU to perform large numbers of address translations. A single 2MB huge page requires a single TLB entry; the same memory, in 4KB pages, would need 512 TLB entries. Given that, it's not surprising that the use of huge pages can make programs run faster. 

The main kernel address space is mapped with huge pages, reducing TLB pressure from kernel code. The only way for user-space to take advantage of huge pages in current kernels, though, is through the hugetlbfs, which was [extensively documented](<https://lwn.net/Articles/374424/>) here in early 2010\. Using hugetlbfs requires significant work from both application developers and system administrators; huge pages must be set aside at boot time, and applications must map them explicitly. The process is fiddly enough that use of hugetlbfs is restricted to those who really care and who have the time to mess with it. Hugetlbfs is often seen as a feature for large, proprietary database management systems and little else. 

There would be real value in a mechanism which would make the use of huge pages easy, preferably requiring no development or administrative attention at all. That is the goal of the transparent huge pages (THP) patch, which was written by Andrea Arcangeli and merged for 2.6.38. In short, THP tries to make huge pages "just happen" in situations where they would be useful. 

Current Linux kernels assume that all pages found within a given virtual memory area (VMA) will be the same size. To make THP work, Andrea had to start by getting rid of that assumption; thus, much of the initial part of the patch series is dedicated to enabling mixed page sizes within a VMA. Then the patch modifies the page fault handler in a simple way: when a fault happens, the kernel will attempt to allocate a huge page to satisfy it. Should the allocation succeed, the huge page will be filled, any existing small pages in the new page's address range will be released, and the huge page will be inserted into the VMA. If no huge pages are available, the kernel falls back to small pages and the application never knows the difference. 

This scheme will increase the use of huge pages transparently, but it does not yet solve the whole problem. Huge pages must be swappable, lest the system run out of memory in a hurry. Rather than complicate the swapping code with an understanding of huge pages, Andrea simply splits a huge page back into its component small pages if that page needs to be reclaimed. Many other operations (`mprotect()`, `mlock()`, ...) will also result in the splitting of a page. 

The allocation of huge pages depends on the availability of large, physically-contiguous chunks of memory - something which Linux kernel programmers can never count on. It is to be expected that those pages will become available at inconvenient times - just after a process has faulted in a number of small pages, for example. The THP patch tries to improve this situation through the addition of a "khugepaged" kernel thread. That thread will occasionally attempt to allocate a huge page; if it succeeds, it will scan through memory looking for a place where that huge page can be substituted for a bunch of smaller pages. Thus, available huge pages should be quickly placed into service, maximizing the use of huge pages in the system as a whole. 

The current patch only works with anonymous pages; the work to integrate huge pages with the page cache has not yet been done. It also only handles one huge page size (2MB). Even so, some useful performance improvements can be seen. Mel Gorman [ran some benchmarks](<https://lwn.net/Articles/423590/>) showing improvements of up to 10% or so in some situations. In general, the results were not as good as could be obtained with hugetlbfs, but THP is much more likely to actually be used. 

No application changes need to be made to take advantage of THP, but interested application developers can try to optimize their use of it. A call to `madvise()` with the `MADV_HUGEPAGE` flag will mark a memory range as being especially suited to huge pages, while `MADV_NOHUGEPAGE` will suggest that huge pages are better used elsewhere. For applications that want to use huge pages, use of `posix_memalign()` can help to ensure that large allocations are aligned to huge page (2MB) boundaries. 

System administrators have a number of knobs that they can tweak, all found under `/sys/kernel/mm/transparent_hugepage`. The `enabled` value can be set to "`always`" (to always use THP), "`madvise`" (to use huge pages only in VMAs marked with `MADV_HUGEPAGE`), or "`never`" (to disable the feature). Another knob, `defrag`, takes the same values; it controls whether the kernel should make aggressive use of [memory compaction](<https://lwn.net/Articles/368869/>) to make more huge pages available. There's also a whole set of parameters controlling the operation of the khugepaged thread; see [Documentation/vm/transhuge.txt](<https://lwn.net/Articles/423592/>) for all the details. 

The THP patch has had a bit of a rough ride since being merged into the mainline. This code never appeared in linux-next, so it surprised some architecture maintainers when it caused build failures in the mainline. Some bugs have also been found - unsurprising for a patch which is this large and which affects so much core code. Those problems are being ironed out, so, while 2.6.38-rc1 testers might want to be careful, THP should be in a usable state by the time the final 2.6.38 kernel is released.
