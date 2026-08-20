---
title: "Hot-page migration and specific-purpose NUMA nodes"
url: https://lwn.net/Articles/1046249/
date: "November 17, 2025"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet November 17, 2025"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 17, 2025

For better or for worse, the NUMA node is the abstraction used by the kernel to keep track of different types of memory. How that abstraction is used, though, is still an active area of development. Two patch sets focused on this problem are currently under review; one addresses the perennial problem of promoting heavily used folios from slower to faster memory, while the other aims to improve the kernel's handling of nodes containing special memory installed for a specific purpose. 

#### Hot-page tracking

Over the last several years, there has been an ongoing push to create systems with multiple classes (or "tiers") of memory. While most memory might be ordinary RAM, a system could also be equipped with a small amount of [high bandwidth memory](<https://en.wikipedia.org/wiki/High_Bandwidth_Memory>) and, perhaps, larger amounts of relatively slow memory. In the absence of other concerns (such as a policy requiring low-price tenants to languish in slower memory), the system is generally best served by placing the most frequently accessed data in its fastest memory. 

Actually placing data that way can be tricky, though. The kernel can, with relative ease, detect memory that has not been accessed for a while; the data stored there can then be pushed down to slower memory. Figuring out which data in slower memory should be moved to faster tiers is harder, but it is an important problem to solve; otherwise demotion to lower tiers could become a one-way trip. Some types of hardware are beginning to gain features that can track frequency of access, but the kernel is not yet prepared to make the best use of those features. 

Bharata B Rao has been working on a solution to this problem for a while; this work was [covered here](<https://lwn.net/Articles/1014220/#hotpage>) in March 2025. The [latest version](<https://lwn.net/ml/all/20251110052343.208768-1-bharata@amd.com>) of his patch series has evolved somewhat since then. At its core, though, it remains a mechanism by which any kernel subsystem that has information about when a given page was last accessed can report that data to a central registry, which then uses the accumulated data to determine which pages (or, more correctly, which folios containing those pages) should be migrated to faster memory. 

This new subsystem provides a call that can be used to report memory activity: 

```
int pghot_record_access(unsigned long pfn, int nid, int src,
                                unsigned long time);
```

The `pfn` parameter should contain the page-frame number of the page that was accessed. If the NUMA node from which the access originated is known, it should be passed in `nid`; otherwise the caller should pick a top-tier node to identify here. The time of the access, if known, is passed in `time`. The `src` parameter describes the source of this information — how the activity on the given page was detected. It should be one of `PGHOT_HW_HINTS` (activity reported by the hardware), `PGHOT_PGTABLE_SCAN` (obtained by scanning page tables), or `PGHOT_HINT_FAULT` (a page fault). 

The way this data is managed has changed in this version of the patch set, which now maintains a single `long int` value for each page being tracked. That is a significant amount of overhead in a kernel where efforts are being made to squeeze every bit out of `struct page`, but it is significantly reduced from previous versions. That integer value is split into four fields containing a NUMA node ID, an access count, the time of last report, and a "should be promoted" flag. 

When a new report comes in, `pghot_record_access()` looks at the existing information for the indicated page. If the NUMA node stored there differs from the node passed to this call, or if the time since the last reported access exceeds a threshold, old data will be discarded and the access count set to one, attributed to the indicated `nid`. Otherwise, the existing access count will be incremented (unless that would cause it to overflow). If the access count exceeds another threshold (wired to two in this series), the sign bit will be set as an indicator that the page is hot and should be moved to a faster tier. 

Also part of this series is a new kernel thread that occasionally scans the hotness data and attempts to promote the pages that have been marked as active. There is a set of sysctl knobs that the administrator can use to control which types of hotness data should be accepted. Another new kernel thread performs page-table scanning, noting which pages are accessed and reporting that information so that the hot pages can be promoted. 

The benchmark results included with the series seem somewhat inconclusive, showing some improvements and some significant regressions. This series might be converging on an approach that will pass muster, but there is still clearly work to be done. 

#### Specific-purpose nodes

As a general rule, if a NUMA node presents memory to the system, that memory becomes freely available for the kernel, and eventually user space, to use as it will. The list of nodes that a process can be allocated can be restricted with the kernel's [memory-policy API](<https://man7.org/linux/man-pages/man2/set_mempolicy.2.html>) or the [cpuset mechanism](<https://man7.org/linux/man-pages/man7/cpuset.7.html>), but the default is to allocate memory globally. That approach does not work well in the presence of memory that has special characteristics that make it unsuitable for general use. Gregory Price is looking to address that problem with the concept of [specific-purpose memory NUMA nodes](<https://lwn.net/ml/all/20251112192936.2574429-1-gourry@gourry.net>), which would host memory that is not generally available for use as system RAM. 

As an example of the type of node he is planning for, he mentions a compressed-memory node. This could be a [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>)-attached device that automatically compresses pages written to it, and decompresses them when they are read back. It would work well as a sort of swap device but, since data stored there is essentially read-only, this memory is not suitable for use as ordinary system RAM. This node could be marked as a specific-purpose node when it is added to the system; [a modified version of zswap](<https://lwn.net/ml/all/20251112192936.2574429-12-gourry@gourry.net>) could allocate pages from it and swap data there, but that node's memory would not be available for most allocation requests. 

The implementation of this concept is simple and clearly not intended to be the complete solution. When added to the system, a memory node can be marked as either "sysram" (ordinary memory) or "specific-purpose"; that designation cannot be changed during the life of the system. The sysram nodes are collected into a list and used to satisfy normal allocation requests. To obtain memory from a specific-purpose node, a kernel function must specify that node explicitly in its allocation request and provide the new `__GFP_SPM_NODE` allocation flag. There is no way to directly allocate specific-purpose memory from user space. 

That is about as far as the series goes; a few things are missing still. For example, while a node can be marked as specific-purpose, there is no way to indicated _which_ of many possible purposes the memory is specific to. The zswap implementation simply takes the first specific-purpose node it finds and assumes it will be of the right type. There has also not yet been effort put into implementing basic memory-management functions like reclaim and compaction on specific-purpose memory. 

The purpose of this patch set is clearly to get a conversation started on how nonstandard types of memory should be handled in the kernel. Price has [a proposed session](<https://lpc.events/event/19/contributions/2143/>) at the upcoming [Linux Plumbers Conference](<https://lpc.events/>) that will try to push some of these ideas forward. Thus, further developments can presumably be expected in December.
