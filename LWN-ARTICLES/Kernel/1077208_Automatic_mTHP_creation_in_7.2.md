---
title: Automatic mTHP creation in 7.2
url: https://lwn.net/Articles/1077208/
date: "June 11, 2026"
category: "Huge pages; Memory management-Huge pages; Releases-7.2"
author: "By Jonathan Corbet June 11, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 11, 2026

The Linux kernel has long tried to use huge pages as a way to improve performance, sometimes with more success than others. The size of huge pages has traditionally been imposed by the hardware, which typically only offers a couple of relatively large options. In more recent times, though, the use of multi-size transparent huge pages (mTHPs), with more flexible sizing implemented in software, has been growing. If all goes well, the 7.2 development cycle will include the addition of [a new feature](<https://lwn.net/ml/all/20260605161422.213817-1-npache@redhat.com>), contributed by Nico Pache, to make the use of mTHPs even more transparent. 

#### A huge-page review

The implementation of traditional huge pages is driven by the system's page-table hierarchy; see [this article](<https://lwn.net/Articles/1064861/>) for details on how that hierarchy works. In short, the smallest huge-page size is obtained by removing the lowest page-table layer (called PTE) for a given entry in the next-higher layer (called PMD) of the hierarchy. On many systems, a PMD-level huge page created in this way is 2MB in size. 

The use of PMD-level huge pages can improve the system's performance in a couple of ways. A huge page can be managed as a unit rather than as 512 individual base pages, reducing memory-management overhead. Each huge page can also be covered by a single translation lookaside buffer (TLB) entry. Those entries are a scarce resource, and using them effectively matters for performance. A virtual-address reference that is resolved via the TLB is vastly faster than one that requires walking the page-table hierarchy, perhaps encountering numerous cache misses on the way. 

Given the performance advantages of huge pages, there has long been a desire to make good use of them; thus the _transparent_ -huge-page feature was added to provide processes with huge pages without the need for any code changes. But PMD-level huge pages have some problems of their own. They are large enough that they can be hard for the kernel to provide after the system has been running long enough to fragment memory. They are also subject to internal fragmentation; if only a small portion of a huge page is actually used by the process that owns it, the rest is simply wasted. Using huge pages in the wrong places can significantly increase a process's total memory use. 

The folio transition has given the kernel a lot more flexibility to manage groups of physically contiguous pages in arbitrary (power-of-two) sizes. When used for a process's memory, larger folios are often referred to as mTHPs. Using mTHPs can, as with PMD-size huge pages, reduce memory-management overhead by reducing the number of folios that must be kept track of. At the same time, mTHPs that are smaller than the PMD size can be easier for the memory-management subsystem to create and allocate; they are also more likely to be fully utilized and less subject to internal fragmentation. More recent processors are capable of using a single TLB entry to cover eight (x86) or 16 (Arm) properly aligned, physically contiguous pages, so managing memory as mTHPs of the correct size can, once again, increase TLB coverage (and, thus, performance). 

#### mTHP collapse support

All this means that large folios (and mTHPs in particular) can improve performance, but only if they are actually used in the right places. The filesystem layers are increasingly good at using large folios for file-backed memory when access patterns suggest that they might help performance. Anonymous memory, though, can be a harder problem. Using mTHPs unconditionally would result in internal fragmentation; they really only make sense when most or all of the base pages within the mTHP are being regularly accessed. 

One part of the implementation of transparent huge pages in the kernel is the `khugepaged` kernel thread; in current kernels, it scans memory in an attempt to join (or "collapse") suitable groups of base pages into PMD-level huge pages. Processes access their anonymous memory in the usual way, faulting in one page at a time (with the kernel perhaps speculatively faulting in nearby pages as well). If a process is fully using a 2MB range of its memory, `khugepaged` may eventually, transparently, substitute one huge page for all of those base pages, a process that normally involves copying the data in those pages to a new location. The process using that memory is none the wiser, other than hopefully experiencing a performance boost. 

`khugepaged` only operates at the PMD level, though. Pache's patch series changes that by allowing `khugepaged` to create mTHPs at other sizes as well. The algorithm used to do this work functions approximately like this, as applied to each 2MB chunk of a process's virtual address space: 

  1. A bitmap is created indicating how many of the base pages within that 2MB region are actually used. In simple terms, a page is deemed to be used if it is present, accessed relatively recently, and contains something other than zeroes. Readers who are interested in the gory details (of which there are many) can look at the implementation of [`collapse_scan_pmd()`](<https://elixir.bootlin.com/linux/v7.1-rc7/source/mm/khugepaged.c#L1251>). 

In current kernels, this scan is aborted for a given 2MB range if too many unused pages (as determined by the [`max_ptes_none` sysctl knob](<https://docs.kernel.org/admin-guide/mm/transhuge.html#:~:text=max%5Fptes%5Fnone%20specifies>)) are found; with too many unused pages, that range is not a candidate for collapsing into a PMD-level huge page. It might still be possible to create mTHPs in that range, though, so Pache's patch set causes the scan to look at all of the pages in the range unconditionally. 
  2. An attempt is made to collapse the full set of pages into a PMD-level huge page, as is done in current kernels. If that attempt succeeds, the job is done. It could fail, though, if more than `max_ptes_none` pages are unused, if a PMD-level huge page cannot be allocated, or for a number of other reasons. 
  3. An attempt is made to collapse pages, starting at the same base, into a smaller mTHP. The size of that mTHP might be half of the previously attempted size, or the kernel could, depending on how it is configured, drop immediately to a smaller size. For example, it might make sense to configure the system to attempt only the PMD size and the size that matches the TLB coalescing done by the processor. The knobs controlling this configuration are documented in [Documentation/mm/transhuge.rst](<https://docs.kernel.org/admin-guide/mm/transhuge.html#global-thp-controls>). 
  4. If the smaller attempt succeeds, `khugepage` will advance its offset beyond the newly created mTHP and restart the process with the largest mTHP size consistent with the alignment of the new address. Otherwise, the target size will be reduced again and control returns to the previous step. 
  5. If the attempt to collapse pages into the smallest possible mTHP fails, the offset is advanced past the failed range of pages, and control returns to step 3\. 

Once a 2MB region has been processed in this way, `khugepaged` moves onto the next 2MB range and restarts from the beginning. If all goes well, the target process's address space will eventually be converted into the largest mTHPs that are consistent with its memory usage and the supply of larger folios. 

#### Avoiding mTHP creep

Creating the largest mTHPs possible is a good goal in general, but there is a problematic failure mode that must be avoided. Imagine a system configured with `max_ptes_none` set to two, meaning that an mTHP will only be created if there are two or fewer unused pages within the range under consideration. On this system, `khugepaged` encounters a range of 16 pages that looks like this: 

> ![\[Range of 16 pages\]](https://static.lwn.net/images/2026/mthp1.svg)

In this diagram, the pages filled in with green are seen to be in use, while those that are gray are unused. When `khugepaged` attempts to create a 16-page mTHP from this range, it will observe four unused pages, so the attempt will fail. Subsequently, though, after `khugepaged` drops back and looks at the beginning of this range as an eight-page mTHP candidate, it will only see two unused pages, so the attempt will succeed, leading to a situation that looks like this: 

> ![\[Range of 16 pages with eight collapsed into an mTHP\]](https://static.lwn.net/images/2026/mthp2.svg)

The lower eight pages have now been collapsed into an mTHP; the same thing will happen to the upper eight pages after `khugepaged` advances its offset. 

Once an mTHP has been created, all of the base pages within it appear to be used if the mTHP itself is used. When, at some future time, `khugepaged` examines this 16-page range again, it will see that all of the pages within it appear to be used; this time, the creation of a 16-page mTHP will proceed. This behavior is referred to as "creep" — the size of the created mTHPs creeps upward with every scan pass until it reaches the PMD size, even if many more than the desired number of base pages within that range are not used by the owning process. 

A fair amount of effort has gone into avoiding creep in the mTHP patch set. Perhaps most visibly, the allowed values of `max_ptes_none` are restricted when creating anything other than PMD-size huge pages. The knob can be set to zero (meaning that an mTHP will only be created if all of the pages in the range look used) or 511, just below the 512 base pages that make up a PMD-size huge page (in which case collapse will always be attempted). Any other value of `max_ptes_none` will result in a warning, with the resulting behavior being as if it were set to zero. Other values of `max_ptes_none` _are_ still implemented as usual for PMD-size huge pages, though. 

This patch series has been through an impressive 19 revisions since [the RFC version](<https://lwn.net/ml/all/20250108233128.14484-1-npache@redhat.com/>) was posted in January 2025. At this point, though, the memory-management developers appear to be happy with it; Lorenzo Stoakes [commented](<https://lwn.net/ml/all/aiMXqZkOJ2dZgyDU@lucifer/>): ""We're good to take this this cycle"". The patches are currently in linux-next and, with luck, will find their way into the mainline in the upcoming merge window. Attention can now, maybe, turn to a rather long list of other huge-page-related patches that have been waiting while this work went through the review process; stay tuned.
