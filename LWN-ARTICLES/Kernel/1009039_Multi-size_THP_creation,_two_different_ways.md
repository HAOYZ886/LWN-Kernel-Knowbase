---
title: "Multi-size THP creation, two different ways"
url: https://lwn.net/Articles/1009039/
date: "February 13, 2025"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet February 13, 2025"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 13, 2025

Huge pages can increase the performance of many programs, but they can also have unfortunate performance impacts of their own. Over the last few years, multi-size transparent huge pages (mTHPs) have increasingly been seen as a happy medium that bring the benefits of huge pages at a lower cost. The system cannot benefit from mTHPs, though, if it does not create them; two developers have independently posted patches to enable the creation of mTHPs in the background. 

#### Huge pages

Huge pages are a feature that is implemented by the CPU and its memory-management unit; they enable the system to run with a mix of page sizes. Those sizes, though, are limited in number and entirely defined by the CPU's memory architecture; they correspond to the levels of the page-table hierarchy. As a reminder, that hierarchy looks vaguely like this diagram (from [this 2013 article](<https://lwn.net/Articles/568076/>)): 

> ![\[Virtual address translation\]](https://static.lwn.net/images/2013/address-translation.png)

(More recent systems can also have [yet another level](<https://lwn.net/Articles/717293/>) below the top of the hierarchy). 

On x86 systems, the base-page size is 4KB. The first level of huge pages is implemented by marking the entry in the page middle directory (PMD) level of the hierarchy — the one just above the bottom page-table entry (PTE) level. The PMD entry then refers directly to a page covering a larger part of the address space rather than to the PTE table. That results in a page size of 2MB; these pages are often referred to as "PMD huge pages". Moving up to the next level directory (the page upper directory, or PUD) yields 1GB huge pages. 

Huge pages that are implemented this way have some significant benefits. A single PMD huge page replaces 512 base pages, significantly reducing the overhead of memory management and the number of page faults the system must handle. It also frees as many as 512 entries in the translation lookaside buffer (TLB), a scarce processor resource that can often be a performance bottleneck. As a result, programs using huge pages can run more quickly. Initially, use of huge pages required special effort on the part of both the developer and the system administrator; the [addition of transparent huge pages (THPs)](<https://lwn.net/Articles/423584/>) in 2011 enabled the kernel to coalesce (or "collapse") a process's base pages into huge pages behind the scenes, making the performance benefits of huge pages available system-wide. 

Unfortunately, huge pages are not entirely free. The cost of huge-page use can be internal fragmentation; if a process only uses part of a huge page, the rest of it is simply wasted. Enabling transparent huge pages can, as a result, significantly increase the memory use of a workload; if a system was running close to its limits without huge pages, turning them on can push it over the edge. For that reason, distributors have often left the feature disabled, and one can easily find advice on the net to leave it that way. 

#### The multi-size option

Multi-size THPs are, as the name would suggest, an extension of the huge-page concept to more sizes — those that do not correspond to the page-table hierarchy. The mTHP concept was enabled by the [transition to folios](<https://lwn.net/Articles/849538/>) as a way of managing memory in the kernel; now it is possible to represent groups of physically contiguous pages of arbitrary (though power-of-two) sizes. Use of mTHPs will, once again, reduce memory-management overhead since they can be managed as a unit rather than as independent base pages. More recent CPUs have added TLB-coalescing features that allow an eight (x86) or 16-page (Arm) mTHP to be represented a single TLB entry, increasing the performance benefit that mTHPs can bring. 

Using mTHPs can thus bring much of the performance improvement seen with traditional huge pages, but with a lower internal-fragmentation cost. It is also much easier for the kernel to allocate an mTHP than a full PMD or PUD huge page, especially after the system has been operating for a while and memory has become fragmented. It is thus not surprising that a fair amount of effort has gone into improving the kernel's ability to use mTHPs in recent years. 

The THP page mechanism can inject huge pages into a process's address space in a couple of ways. One of those is when the pages are initially allocated. Rather than satisfying a page fault with a base page, the kernel can allocate a huge page and read all of the contents in a single operation, for example. The other is to examine a process's address space in the background and identify ranges of virtually contiguous pages that could be replaced with huge pages. When such a range is found, a huge page is allocated, the base pages are copied into it, then freed after the huge page replaces them in the page tables. This latter task is performed by the `khugepaged` kernel thread, which benefits from being able to observe which pages of memory a process is actually using. The `khugepaged` thread, though, can only create PMD huge pages; it predates mTHPs and does not create them. The two patch sets in question are both aimed at filling in that gap. 

Before looking into them, there is one other bit of useful background relevant to both approaches. There is a sysctl knob, `max_ptes_none`, that controls how many new pages `khugepaged` can allocate to create a huge page. If, for example, a suitable range of address space contains 500 active pages, `khugepaged` would have to allocate 12 more pages to have the full 512 that make up a PMD huge page. If `max_pte_none` is set to at least 12, `khugepaged` will proceed, otherwise it will give up on that huge page. Creating mTHPs in `khugepaged` increases the amount of thought that must be put into this knob's setting. 

#### Improving `khugepaged`

The [first patch set](<https://lwn.net/ml/all/20250211003028.213461-1-npache@redhat.com>), from Nico Pache, changes the core `khugepaged` scanning loop (called [`hpage_collapse_scan_pmd()`](<https://elixir.bootlin.com/linux/v6.14-rc2/source/mm/khugepaged.c#L1267>) in current kernels, though the series renames it to `khugepaged_scan_pmd()`). The current implementation will stop scanning a candidate huge page immediately if `max_pte_none` unmapped pages are found. In the modified version, instead, it will scan the entire address range for the huge page regardless and create a bitmap marking the places where an mTHP could be created within that page. The smallest size considered is order 3 (eight pages); that may change to allow an order 4 minimum on Arm systems in the future, matching the size at which TLB entries can be coalesced. 

When determining whether a given mTHP-size chunk could be turned into an actual mTHP, the patch scales the `max_pte_none` value accordingly. When looking at an eight-page chunk, `max_pte_none` will be divided by 64 (512/8) to determine how many pages must actually be present. 

After a PMD has been scanned, `khugepaged` will first consider whether the whole thing can be turned into a PMD huge page. Failing that, the bitmap created during the scan will be consulted in an attempt to create the largest mTHPs possible given the pages that are actually resident in memory. Once the bitmap has been processed and any mTHPs created, `khugepaged` will move on to the next PMD to scan. 

There is one problem with this approach that both patch sets must address; it has been called "creep". Imagine that `max_pte_none` has been set to 511, which happens to be its default value. At that setting, even a single resident page is enough to cause `khugepaged` to populate the rest of a huge page, which may well be what the user wants. Things change a bit when dealing with mTHPs, though. When a PMD is scanned, each single page will lead to the creation of an eight-page mTHP, as expected. The next time the scan happens, though, those mTHPs will each be turned into 16-page mTHPs, since each contains enough pages to cause `khugepaged` to bump them up to the next size. 

Over time, these mTHPs will all "creep" up to the PMD size, which is just the result that the patch set was created to avoid. To keep this from happening, administrators will need to set `max_pte_none` to a value that is less than half of the PMD huge page size — a value of 255 or smaller. 

The [second patch series](<https://lwn.net/ml/all/20250211111326.14295-1-dev.jain@arm.com>), from Dev Jain, takes a slightly different approach. `khugepaged` begins by scanning in PMD-sized chunks as before, but if it is unable to collapse a given PMD range, it drops down to a smaller order and tries again. The [code implementing this algorithm](<https://lwn.net/ml/all/20250211111326.14295-8-dev.jain@arm.com>), being a mass of `goto` statements, is a bit hard to follow, but the intent is to scan each range of memory at the size that results in successful mTHP creation. Once a PMD entry has been completed, the order is reset to the maximum to start the next one. 

This series takes a stricter approach to the creep problem; it will only perform mTHP collapse if `max_ptes_none` is set to either zero or 511\. In the former case, mTHPs will only be created if all of the base pages it would replace are already present; in the latter case, creep up to the full PMD size is explicitly allowed. 

Pache posted [another patch set](<https://lwn.net/ml/all/20250211004054.222931-1-npache@redhat.com>) that could conceivably be used with either of the `khugepaged` mTHP implementations. In current kernels, mTHPs of various sizes can be explicitly enabled or disabled by way of a set of sysctl knobs (documented in [`Documentation/admin-guide/mm/transhuge.rst`](<https://docs.kernel.org/admin-guide/mm/transhuge.html#sysfs>)). The setting of `always` causes the kernel to try to create mTHPs of that size, while `never` disables that size. Pache adds a new setting of `defer` that will cause the kernel to _not_ use the given size in response to page faults, but _will_ allow `khugepaged` to collapse to that size. The purpose here is to limit mTHP allocations to memory where it has been shown to be useful, hopefully further reducing the costs of mTHP usage while enjoying their benefits. 

Both of these patch sets are freshly posted as of this writing; neither has yet generated any real review discussion beyond [a request](<https://lwn.net/ml/all/20250211152341.3431089327c5e0ec6ba6064d@linux-foundation.org>) for benchmark results from Andrew Morton. So there is no way to predict which (if either) of the patch sets will be accepted, or how much change may be needed before that can happen. There is a clear desire to make better use of mTHPs, though, so the goal behind this work is easy enough to understand; sooner or later, the kernel will almost certainly gain some sort of ability to create mTHPs in the background while a process is running.
