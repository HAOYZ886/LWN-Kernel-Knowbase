---
title: "Process-level kernel samepage merging control"
url: https://lwn.net/Articles/928510/
date: "April 13, 2023"
category: "Memory management-Kernel samepage merging"
author: "By Jonathan Corbet April 13, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 13, 2023

The [kernel samepage merging (KSM)](<https://www.kernel.org/doc/html/latest/admin-guide/mm/ksm.html>) feature can save significant amounts of memory with some types of workloads, but security concerns have greatly limited its use. Even when KSM can be safely enabled, though, the control interface provided by the kernel makes it unlikely that KSM actually will be used. A small [patch series](<https://lwn.net/ml/linux-mm/20230406165339.1017597-1-shr@devkernel.io/>) from Stefan Roesch aims to change this situation by improving and simplifying how KSM is managed. 

As its name would suggest, KSM works by finding pages of memory with identical contents and merging them into a single copy that is shared by all users. An early use case, as [described by Avi Kivity](<https://lwn.net/Articles/306704/>) in 2008 when the feature was first proposed, was ""the typical multiuser gnome minicomputer with all 150 users reading lwn.net at the same time instead of working""; this workload would generate a lot of identical cache pages that could be shared rather than duplicated across the system. There are other use cases, such as virtual machines or containers running the same software, that could also be optimized once the important workloads have been addressed. 

There can be value in performing this kind of deduplication. Some workloads, it turns out, produce a lot of identical pages; merging those pages cuts the memory use of those workloads considerably, allowing more work to be crammed into the system. KSM thus looked appealing. The merging of this feature [was delayed](<https://lwn.net/Articles/330589/>) for some time as the result of security and patent concerns, but it finally made it into the 2.6.32 release at the end of 2009. It turns out, though, that there were more security problems inherent in this mechanism than had been originally thought. 

KSM works by scanning pages in memory to locate pages with the same contents. When such pages are found, all users are made to share a single copy of that page, while the duplicate copies are returned to the system's free list. Some care must be taken, though: KSM works with anonymous pages (file-backed pages are already shared via the page cache), and the owners of those pages can change their contents at any time. One process's change should, clearly, not affect any other processes that might be sharing that page. To ensure correct behavior, the kernel marks shared pages as read-only; should a process write to such a page, a copy-on-write (COW) operation will be performed to give the writing process its own copy again. With this mechanism in place, pages can be safely shared between processes that do not trust each other, with those processes being entirely unaware of the tricks going on behind their back. 

At least, that was the intent. A write to an exclusively owned page is a fast operation, while a write to a read-only page that forces a COW operation takes quite a bit longer. A hostile process can use this timing difference to determine whether a page had been shared by the kernel or not; that, in turn, allows the attacker to determine whether a page with specific contents exists in the system. Given enough time, these timing attacks can be used to find out whether a specific program is running on the system, determine another process's address-space layout randomization offsets, or even exfiltrate cryptographic keys. 

The security concerns associated with KSM are sufficiently worrying that the feature is not generally enabled. The only way to turn it on is for a process that wishes to participate in sharing to make an [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) call (`MADV_MERGEABLE`) to enable KSM for a specific memory range. Few programs do this, so KSM is not widely used. As Roesch notes in the cover letter, even developers who are aware of KSM and wish to enable it may be unable to if they are working in a garbage-collected language that does not provide access to `madvise()`. 

The proposed solution to this problem is a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) operation (`PR_SET_MEMORY_MERGE`) that sets the KSM status for a process as a whole. If this operation is used to turn on KSM, every virtual memory area (VMA) within the process that could be merged will have KSM enabled, as will any eligible VMAs created after the call. This setting is inherited by any child processes so, for example, if the first process for a control group or virtual machine enables KSM, all descendant processes will have it enabled as well. 

This feature allows an orchestration system for virtual machines or containers to enable KSM for the workloads it launches; the use of KSM will no longer be something that individual processes must opt into. The orchestration system may know enough about the workloads it runs to be able to determine whether KSM can be enabled safely; individual processes generally lack that view. Systems where KSM can be enabled in this way, Roesch said, have ""shown a capacity increase of around 20%"". In other words, the system described by Kivity 15 years ago would now be able to support 180 unproductive LWN readers, which seems like an improvement for everybody involved. 

The patch set makes a few other changes as well, mostly aimed at improving the effectiveness metrics produced by KSM, so administrators can determine whether the run-time overhead of scanning pages is justified by the resulting memory savings. 

This patch set is currently in its fifth revision. The comments on previous postings have mostly been concerned with where the memory savings are coming from. In the early days of KSM, the biggest win came from workloads that kept a lot of zero-filled pages around, but that does not appear to be the case anymore; KSM is deduplicating a lot of non-zero pages. There does not appear to be much that would block this series from landing upstream in the near future.
