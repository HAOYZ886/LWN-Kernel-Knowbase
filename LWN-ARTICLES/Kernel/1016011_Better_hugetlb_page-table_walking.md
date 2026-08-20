---
title: "Better hugetlb page-table walking"
url: https://lwn.net/Articles/1016011/
date: "April 3, 2025"
category: "Memory management-Internal API"
author: "By Jonathan Corbet April 3, 2025 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 3, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The kernel must often step through the page tables of one or more processes to carry out various operations. This "page-table walking" tends to be performed by ad-hoc (duplicated) code all over the kernel. Oscar Salvador used a memory-management-track session at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit to talk about strategies to unify the kernel's page-table walking code just a little bit by making hugetlb pages look more like ordinary pages. 

[![\[Oscar Salvador\]](https://static.lwn.net/images/conf/2025/lsfmm/OscarSalvador-sm.png)](<https://lwn.net/Articles/1016012/>) "Hugetlb" refers to an early huge-page implementation in the kernel that has often been thought of as an independent memory-management subsystem. It works with memory that has been reserved specially; the hugetlbfs filesystem must be used to gain access to that memory. Many applications are better served by transparent huge pages, which require no special code, but hugetlb users remain. It gives more reliable access to huge pages for some applications, and can reduce memory usage by sharing page tables across multiple processes. 

The existence of hugetlb as an independent memory-management mechanism has long grated on the development community. The 2024 Summit featured [a session focused on hugetlb unification](<https://lwn.net/Articles/974491/>), and some progress has been made in that direction. The 2025 session limited its scope to page-table walking in particular, in the hope of getting rid of some duplicated code and special cases. Salvador posted [an RFC patch set](<https://lwn.net/ml/all/20240704043132.28501-1-osalvador@suse.de/>) unifying the hugetlb page-walking code in July 2024, but the reviews were mixed, and that work has not proceeded further. 

Since then, David Hildenbrand has proposed, in general terms, a new page-walking API that could be considered instead. (That initial proposal happened in [this email](<https://lwn.net/ml/all/74ecaa8b-9e94-4ba8-a2f0-a312607516ba@redhat.com/>), but most of the discussion about the implementation has evidently happened privately. Salvador has an implementation in [his repository](<https://github.com/leberus/linux/commit/e4144766017fab44f3671a53ccde16f1dd8f8d94>).) The core idea is an API that walks through a virtual-memory area (VMA) and manages locking and batching of operations, telling the caller what type of pages were found. This new API would make implementation of `/proc/_PID_ /smaps` much simpler, he said. If the group agreed, he said he would like to start converting some of the `/proc` code over, then move on to some of the other page-table walkers in the kernel. 

Lorenzo Stoakes asked whether Salvador intended to replace all of the page-table walkers in the kernel with calls to the new API. Salvador said he did not intend to do that right away; there are a lot of special cases in many of those walkers, so the conversion is not always straightforward. Hildenbrand said that, for now, it is best to focus on the lower levels of the page tables. 

Ryan Roberts expressed concerns that the performance of many system calls is sensitive to small changes. Adding a page-table walker with indirect calls could introduce an unacceptable slowdown, he said. But, as it turns out, the proposed API is implemented as an iterator with no indirect calls, so that should not be a problem. At the end of the session, Matthew Wilcox asked how this API will handle a copy-on-write operation in the middle of a large allocation. For now, apparently, it does not handle that case at all; in the future, it will be able to return ranges of compatible page-table entries.
