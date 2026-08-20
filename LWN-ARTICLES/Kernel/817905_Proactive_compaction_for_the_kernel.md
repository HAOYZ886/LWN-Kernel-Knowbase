---
title: Proactive compaction for the kernel
url: https://lwn.net/Articles/817905/
date: "April 21, 2020"
category: "Memory management-Large allocations"
author: "April 21, 2020 This article was contributed by Nitin Gupta"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

April 21, 2020

This article was contributed by Nitin Gupta

Many applications benefit significantly from the use of huge pages. However, huge-page allocations often incur a high latency or even fail under fragmented memory conditions. Proactive compaction may provide an effective solution to these problems by doing memory compaction in the background. With my proposed proactive compaction implementation, typical huge-page allocation latencies are reduced by a factor of 70-80 while incurring minimal CPU overhead. 

[Memory compaction](<https://lwn.net/Articles/368869/>) is a feature of the Linux kernel that makes larger, physically contiguous blocks of free memory available. Currently, the kernel uses an on-demand compaction scheme. Whenever a per-node `kcompactd` thread is woken up, it compacts just enough memory to make available a single page of the needed size. Once a page of that size is made available, the thread goes back to sleep. This pattern of compaction often causes a high latency for higher-order allocations and hurts performance for workloads that need to burst-allocate a large number of huge pages. 

Experiments where compaction is manually triggered on a system with a fragmented memory state show that it could be brought to a fairly compacted memory state within one second for a 32GB system. Such data suggests that a proactive compaction scheme in the kernel could allow allocating a significant fraction of memory as huge pages while keeping allocation latencies low. 

My recent [patch](<https://lwn.net/ml/linux-kernel/20200310222539.1981-1-nigupta%40nvidia.com/>) provides an implementation of the proactive compaction approach. It exposes a single tunable: `/sys/kernel/mm/compaction/proactiveness`, which accepts values in the range [0, 100], with a default value of 20. This tunable determines how aggressively the kernel should compact memory in the background. The patch reuses the existing, per-NUMA-node `kcompactd` threads to do background compaction. Each of these threads periodically calculates a per-node fragmentation score (an indicator of the memory fragmentation) and compares it against a threshold, which is derived from the tunable. 

The per-node proactive (background) compaction process is started by its corresponding `kcompactd` thread when the node's fragmentation score exceeds the high threshold. The compaction process remains active till the node's score falls below the low threshold, or one of the back-off conditions (defined below) is met. 

Memory compaction involves bringing together "movable" pages at a zone's end to create larger, physically contiguous, free regions at a zone's beginning. If there are "unmovable" pages, like kernel allocations, spread across the physical address space, this process is less effective. Compaction has a non-trivial system-wide impact as pages belonging to different processes are moved around, which could also lead to latency spikes in unsuspecting applications. Thus, the kernel must not overdo compaction and should have a good back-off strategy. 

The patch implements back-offs in the following scenarios: 

  * When the current node's `kswapd` thread is active (to avoid interfering with the reclaim process).
  * When there is contention on the per-node [`lru_lock`](<https://elixir.bootlin.com/linux/v5.6/source/include/linux/mmzone.h#L769>) or per-zone [`lock`](<https://elixir.bootlin.com/linux/v5.6/source/include/linux/mmzone.h#L536>) (to avoid hurting non-background, latency-sensitive contexts).
  * When there is no progress (reduction in node's fragmentation score value) after a round of compaction.

If any of these back-off conditions is true, the proactive compaction process is deferred for a specific time interval. 

#### Per-node fragmentation score and threshold calculation

As noted earlier, this proactive compaction scheme is controlled by a single tunable called proactiveness. All required values like per-node fragmentation score and thresholds are derived from this tunable. 

A node's score is in the range [0, 100] and is defined as the sum of all the node's zone scores, where a zone's score (Sz) is defined as: 

```
Sz = (Nz / Nn) * extfrag(zone, HUGETLB_PAGE_ORDER)
```

where: 

  * `Nz` is the total number of pages in the zone.
  * `Nn` is the total number of pages in the zone's parent node.
  * `extfrag(zone, HUGETLB_PAGE_ORDER)` is the external fragmentation with respect to the huge-page order in this zone.

In general, this is the way to calculate external fragmentation with respect to any order: 

```
extfrag(zone, order) =  ((Fz - Hz) / Fz) * 100
```

where: 

  * `Fz` is the total number of free pages in the zone.
  * `Hz` is the number of free pages available in blocks of size >= 2order.

This per-zone score value is in the range [0, 100], and is defined as 0 when `Fz = 0`. The reason for calculating the per-zone score this way is to avoid wasting time trying to compact special zones like `ZONE_DMA32` and focus on zones like `ZONE_NORMAL`, which manage most of the memory (`Nz ≈ Nn`). For smaller zones (`Nz << Nn`), the score tends to 0, and thus can never cross the low threshold value (defined below). 

The low (`Tl`) and high (`Th`) thresholds, against which these scores are compared, are defined as follows: 

```
Tl = 100 - proactiveness
        Th = min(10 + Tl, 100)
```

These thresholds are in the range [0, 100]. Once a zone's score exceeds `Th`, proactive compaction will be done until the score drops below `Tl`. 

#### Performance evaluation

For a true test of memory compaction efficacy, the first step is to fragment the system memory such that no huge pages are directly available for allocation (i.e., an initial fragmentation score of 100 for all NUMA nodes). With the system in such a fragmented memory state, any run of a huge-page-heavy workload would highlight the effects of the kernel's compaction policy. With on-demand compaction, a majority of huge-page allocations hit the direct-compaction path, leading to high allocation latencies. With proactive compaction, the expectation is to avoid these latencies except when the compaction process is unable to catch up with the huge-page allocation rate. 

For evaluating proactive compaction, a high level of external fragmentation (as described above) was triggered by a user-space program that allocated almost all memory as huge pages and then freed 75% of pages from each huge-page aligned chunk. All the tests were done on an x86_64 system with 1TB of RAM and two NUMA nodes, using kernel version 5.6.0-rc4. The first huge-page-heavy workload was a test kernel driver that allocated as many huge pages as possible, measuring latency for each allocation, until it hit an allocation failure. The driver was able to allocate almost 98% of free memory as huge pages with or without the patch. However, with the vanilla kernel (without the proactive compaction patch), 95-percentile latency was 33799µs while with the patch (with proactiveness tunable set to 20), this latency was 429µs -- a reduction by a factor of 78. 

To further measure the performance, a Java heap allocation test was used, which allocated 700GB of heap space, after fragmenting the memory as described above. This workload shows the effect of reduced allocation latencies in the run time of huge-page-heavy workloads. On the vanilla kernel, the run time was ~27 minutes, while with the patch (proactiveness=20), the run time came down to roughly four minutes. Tests with higher proactiveness values did not show any further speedups or slowdowns. 

Some [questions](<https://lwn.net/ml/linux-kernel/ae306f4d-4dff-a97b-00b1-71d7ab54f68b%40suse.cz/>) were raised by Vlastimil Babka, who has [worked on proactive compaction](<https://lwn.net/Articles/717656/>) along the way and helpfully reviewed some of these patches: 

By your description it seems to be a one-time fragmentation event followed by a one-time stream of allocations? So kcompactd probably did the proactive work just once? That works as a smoke test, but what I guess will be more important is behavior under more complex workloads, where we should also check the vmstat compact_daemon* stats and possibly also kcompactd kthreads CPU utilizations. 

The comment led to further investigation into the proactive `kcompactd` behavior. For the Java heap test, per-node fragmentation scores were recorded during the program's runtime, together with `kcompactd` threads' CPU usage. The data clearly shows that proactive compaction is active throughout the runtime of the test program and that a `kcompactd` thread takes 100% of one of the CPU cores while it is active.

> ![\[Proactive compaction performance graph\]](https://static.lwn.net/images/2020/proactive-compaction.svg)

The above plot shows changes in per-node fragmentation scores as a Java process tries to allocate 700GB of the heap area on a two-node system with 512GB memory each. The proactiveness tunable is set to 20, so the low and high thresholds are 80 and 90, respectively. Before the test program was run, the system memory was fragmented, such that no huge pages were available for direct allocation. The heap allocation started on Node-0, where the score rose as huge pages are used. When the score gets above 90, proactive compaction is triggered on the node, bringing the score back to 80\. Eventually, all memory on Node-0 was exhausted (around 90 seconds into the run), at which point the allocation started from Node-1, where the same compaction pattern repeated.

As described earlier, compaction is an expensive operation, and we don't want to pay the cost unless it is able to reduce the external fragmentation. To evaluate the back-off behavior, another test was created where unmovable pages were scattered throughout the physical address space. With the system in such a memory state, compaction cannot do much apart from warming the CPU. The back-off mechanism implemented in this patch correctly identifies such a situation by checking that a node's score does not go down after a round of compaction. When that happens, further rounds of proactive compaction are deferred for some time. 

#### Future work

The patch has already been through some review cycles, which have helped refine many of its aspects. Some kernel developers recognize the need for a more proactive compaction, and given these encouraging numbers, the patch will hopefully be accepted, probably after a few more iterations. 

As a future direction, I am focusing on refining the per-node fragmentation score and threshold calculations, which drive the background proactive compaction process. Currently, the score calculation does not take into account per-node characteristics like differing TLB latencies, which would be important in heterogeneous systems. Future patches will likely add scaling factors to both score and threshold calculations to account for these per-node characteristics.
