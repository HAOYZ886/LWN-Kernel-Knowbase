---
title: "Improving hot-page detection and promotion"
url: https://lwn.net/Articles/1016519/
date: "April 9, 2025"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet April 9, 2025 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 9, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Tiered-memory systems feature multiple types of memory with varying performance characteristics; on such systems, good performance depends on keeping the most frequently used data in the fastest memory. Identifying that data and placing it properly is a challenge that has kept developers busy for years. Bharata Rao, presenting remotely during a memory-management-track session at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, led a discussion on [a potential solution](<https://lwn.net/Articles/1014220/#hotpage>) he has recently posted; Raghavendra K T was also named on [the session proposal](<https://lwn.net/ml/linux-mm/20250123105721.424117-1-raghavendra.kt@amd.com/>). It seems likely, based on the discussion, that developers working in this area will not run out of problems anytime soon. 

There are two aspects to the memory-promotion problem: detecting which pages should be moved, and actually migrating them. On the detection side, there are a number of sources for data that can be used to detect hot (frequently accessed) pages; Rao is mostly focused on approaches that involve scanning page tables to see which pages have been accessed recently. He thinks that the current scanning implementation needs an overhaul; it is tied to the NUMA-balancing code, and its operation tends to create latency spikes for applications. Since both the page-table-entry (PTE) scanning and page migration are done in process context, they interrupt user-space execution in potentially disruptive ways. 

Additionally, he said, there is an increasing number of ways to detect memory access — new data-temperature sources are becoming available. That suggests a need for a way to gather this data together and act on it in one place. His proposal is a new kernel thread, `kmmscand`, that would take over this task. It would maintain a list of process address spaces, then perform "A-bit" scanning (clearing the "accessed" bit in the PTEs, then scanning later to see which PTEs have had the bit set, indicating an access in the meantime) on each. The results of the scan are then used to create a list of pages to promote to faster memory. 

In Rao's implementation, the migration task is separated into its own thread that runs independently of the scanning thread. The performance impact of the scanning can be regulated by adjusting the rate at which PTEs are scanned or by exempting some processes from scanning altogether. There is still room for improvement, though, he said. In particular, there is always a need for better hot-page detection, and there is a need for throttling and batching of the migration work as well. 

A participant asked how often address spaces are scanned; Rao answered that, initially, a scan is performed every two seconds. Over time, the rate is adjusted according to conditions; it can end up anywhere between 0.5 and five seconds. 

Another participant said that effective scanning requires flushing the translation lookaside buffer (TLB); otherwise, it will short out the address-translation process that sets the A bit in the first place. TLB flushes, of course, can hurt performance. Rao answered that the current implementation promotes pages the first time an access is observed, so missing data on subsequent accesses is not a problem. SeongJae Park said that [DAMON](<https://docs.kernel.org/admin-guide/mm/damon/>), which also performs this sort of scanning, is not currently doing TLB flushes. That drew another question about whether this scanning should just be integrated into DAMON rather than being implemented as a separate thread; Rao answered that integrating the systems is something he will be looking into. Davidlohr Bueso pointed out that enterprise distributions do not normally enable DAMON, but Park answered that Red Hat and Debian are indeed enabling it. 

Gregory Price said that the scanning thread will have to hold locks while passing over the page tables, and asked whether that has been observed to cause latencies for the processes being scanned. Rao said that the overall result of his implementation is a significant reduction in latencies experienced by applications, but did not address the locking issue specifically. Jonathan Cameron asked how the scanning of huge pages was handled; Rao said that the thread is just scanning PTEs and not doing anything special for huge pages. 

Another participant worried about the cost of scanning especially large address spaces and asked if the overhead had been measured. Rao answered that there is still a need to understand the full cost of A-bit scanning. If it turns out to be a problem, there are optimizations, such as scanning at the PMD level first before dropping down to the PTE level in hot areas, that can be considered. There have been some experiments done with 64GB address spaces, he said, that have shown an improvement over current kernels. 

A remote participant pointed out that the [multi-generational LRU](<https://lwn.net/Articles/856931/>) also optimizes its scanning by looking at higher-level page tables first; among other things, that helps it to avoid scanning unmapped memory. Cameron pointed out that scanning the higher-level tables will only be useful if the TLB is regularly flushed. But Yuanchu Xie said that the multi-generational LRU scanning seems to be optimized well, and is integrated with the reclaim system as well. Rao agreed that this avenue needs more exploration. 

At the end of the session, Rao raised one last problem: while A-bit scanning can identify hot pages, it provides no information about which NUMA nodes have accessed any given page. As a result, there is no way to know which node a hot page should be promoted to. There needs to be a way to maintain home-node information for pages he said. Alternatively, the system could also scan pages in the fastest tiers to see where the pages in a given address space are clustered now, then promote other pages to the same nodes. The group seemed to think that this could be an interesting heuristic to explore. 

Rao has [posted the slides](<https://docs.google.com/presentation/d/1zLyGriEyky_HLJPrrdKdhAS7h5oiGf4tIdGuhGX3fJ8/edit#slide=id.g3435fddbc8d_2_236>) from this session.
