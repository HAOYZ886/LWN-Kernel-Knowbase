---
title: Weighted interleaving for memory tiering
url: https://lwn.net/Articles/948037/
date: "October 25, 2023"
category: "Memory management-Tiered-memory systems; Releases-6.9"
author: "By Jonathan Corbet October 25, 2023"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 25, 2023

The kernel has, for many years, had the ability to control how memory allocation is performed in systems with multiple NUMA nodes. More recently, NUMA nodes have also been pressed into service to represent different classes of memory; those nodes are now organized into tiers according to their performance characteristics. While memory-allocation policies can control the placement of pages at the NUMA-node level, the kernel provides no way to connect those policies with memory tiers. [This patch series](<https://lwn.net/ml/linux-kernel/20231009204259.875232-1-gregory.price@memverge.com/>) from Gregory Price aims to change this situation by allowing allocations to be placed across tiers in a weighted manner. 

The days when computers simply had "memory" have passed. Systems now can contain numerous types of memory with wildly different performance profiles. Persistent memory can be available in large quantities, but it is relatively slow. [CXL memory](<https://lwn.net/Articles/894598/>) may be faster, though still not as fast as normal DRAM, and it can come and go during the life of the system. High-bandwidth memory can be faster than normal DRAM. Devices, too, can make their own memory available on the system's memory bus. Each memory provider is represented in the system as one or more CPU-less NUMA nodes. 

Tiering is the process of grouping these nodes into levels (tiers) with similar performance, then working to place memory optimally in each tier; it is a work in progress. The kernel only gained a formal concept of memory tiers in the 6.1 release, when [this series](<https://lwn.net/ml/linux-kernel/20220818131042.113280-1-aneesh.kumar@linux.ibm.com/>) from Aneesh Kumar K.V. was merged. The kernel now assigns each node an "abstract distance" reflecting its relative performance; by default, all nodes have an abstract distance of 512. Memory with a lower abstract distance than that is expected to be faster than ordinary DRAM, while a higher abstract distance indicates slower memory. Notably, the distance is set by the driver that makes the memory available to the system; it is not subject to direct administrator control. 

As memory is initialized, the kernel will organize it into distinct tiers, with the tier number for any given node obtained by dividing its abstract distance by 128\. Normal memory, with its default abstract distance of 512, ends up in tier four as a result. There is, thus, room for four tiers of memory that is faster than DRAM, and a vast number of slower tiers. 

The kernel can use tiering to make decisions about page placement; [quite a bit of work](<https://lwn.net/Articles/893024/>) has gone into trying to figure out how to move pages between the tiers. When pages are unused, they can be demoted to slower tiers rather than being reclaimed entirely; that is a relatively easy decision that fits into existing memory-management practices. Deciding when pages should be promoted back to faster memory, though, is a bit trickier. Importantly, this work has been focused on moving memory to the correct tier after it has been in use for a while and its relative busyness can be assessed. 

Price's patch set builds on the tiering mechanism, but is aimed at a different problem: what is the optimal _original_ placement for pages when they are first allocated? In particular, it addresses the practice of NUMA interleaving, wherein allocations are spread out across a set of NUMA nodes with the purpose of getting consistent performance from any of a set of the CPUs on the system. NUMA interleaving has worked for years, but it is relatively simple; it spreads pages equally across the set of nodes provided in the memory policy set by the application. Interleaving has no concept of tiers and no ability to bias allocations in any specific direction. 

The patch series adds the concept of a "weight" to each NUMA tier. Weights are relative to each CPU, so the weight of tier 2 as seen from CPU 0 may be different than the weight of that tier seen from CPU 10\. These weights determine how strongly allocation decisions should be biased toward each tier; a higher weight for a given tier results in more pages being allocated on nodes contained within that tier. Unlike the abstract distance, the tier weights are controlled by the administrator; a command like: 

```
echo 0:20 > /sys/devices/virtual/memory_tiering/memory_tier2/interleave_weight
```

would set the weight of tier 2, as seen by CPU 0, to 20\. By default, all tiers have a weight of one. 

These weights are used by the NUMA interleaving code when pages are allocated. If a system has two tiers, tier 4 (where normal DRAM ends up) and tier 8 (with slower memory), and the weights of those tiers for the current CPU are set to 20 and 10 respectively, then interleaved allocations will place twice as many pages on tier 4 as on tier 8\. It's worth noting that, while the application specifies its NUMA allocation policy (which nodes it wants to allocate memory on), the weights are solely under the control of the administrator. 

While promotion and demotion between tiers is aimed at putting the most-used pages on the fastest memory, weighted allocation has a different objective — necessarily, since the kernel cannot know at allocation time which pages will be the most heavily used. The intent, instead, is to try to obtain an optimal split of memory usage across memory controllers, maximizing the bandwidth utilization of each. Moving some pages from faster memory to a slower tier can result in an overall performance increase, even if the moved pages are actively used, if the end result is to keep both tiers busily moving data. 

According to Price, this approach works: 

> Observed a significant increase (165%) in bandwidth utilization with the newly proposed multi-tier interleaving compared to the traditional 1:1 interleaving approach between DDR and CXL tier nodes, where 85% of the bandwidth is allocated to DDR tier and 15% to CXL tier with MLC -w2 option. 

(MLC refers to Intel's proprietary [memory latency checker](<https://www.intel.com/content/www/us/en/developer/articles/tool/intelr-memory-latency-checker.html>) tool). 

This is the second version of this patch set (the first was [posted by Ravi Jonnalagadda](<https://lwn.net/ml/linux-kernel/20230927095002.10245-1-ravis.opensrc@micron.com/>) in late September). There have been a number of comments about the details of the implementation, but there does not appear to be any opposition to the concept in general. Since all tiers start with an identical weight, there will be no changes to NUMA interleaving unless the administrator has explicitly changed the system's configuration. There would thus appear to be little reason for this work not to advance once the details have been taken care of.
