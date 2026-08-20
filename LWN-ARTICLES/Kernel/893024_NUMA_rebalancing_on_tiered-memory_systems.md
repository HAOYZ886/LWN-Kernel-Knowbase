---
title: "NUMA rebalancing on tiered-memory systems"
url: https://lwn.net/Articles/893024/
date: "May 2, 2022"
category: "Memory management-NUMA systems; Memory management-Tiered-memory systems"
author: "May 2, 2022 This article was contributed by David Vernet"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

May 2, 2022

This article was contributed by David Vernet

The classic NUMA architecture is built around nodes, each of which contains a set of CPUs and some local memory; all nodes are more-or-less equal. Recently, though, "tiered-memory" NUMA systems have begun to appear; these include CPU-less nodes that contain persistent memory rather than (faster, but more expensive) DRAM. One possible use for that memory is to hold less-frequently-used pages rather than forcing them out to a backing-store device. There is an interesting problem that emerges from this use case, though: how does the kernel manage the movement of pages between faster and slower memory? Several recent patch sets have taken differing approaches to the problem of rebalancing memory on these systems. 

#### Migration and reclaim

The kernel detects if a given page needs to be migrated using a technique called "NUMA hint-faulting". Ranges of a task's address space are periodically unmapped so that subsequent accesses to a page in the range will trigger a page fault. When a page fault occurs, the memory management subsystem can then use the location of the CPU that triggered the page fault to determine whether the page needs to be migrated to the node which contains that CPU. The absence of a fault altogether indicates that the page is getting colder, and may be migrated to a slow-tier node during reclaim. As workloads run and access patterns change, pages transition between hot and cold, and are migrated between fast and slow NUMA nodes accordingly. 

Memory reclaim is driven by a "watermark" system that tries to keep at least a minimum number of free pages available. When an allocation is requested, the kernel compares the number of free pages in the node where the allocation is taking place to a zone watermark threshold. If the number of free pages in the node, after the allocation, is lower than the threshold specified by the watermark, then the `kswapd` kernel thread is awoken to asynchronously scan and reclaim pages from the node. This allows memory to be freed preemptively, before memory pressure in the node causes allocations to block, and direct reclamation to occur. 

Zone watermarks in the kernel are statically sized according to the memory profile of the host. Systems with less memory will have lower zone watermarks, while watermarks for larger systems will be higher. Intuitively, this scaling makes sense. If you have a machine with a huge amount of memory, reclaim should probably be triggered sooner than on a machine with very little memory, as the expectation is that an application will be more aggressive in requesting memory on a system that has more of it. Yet, having a static threshold also has drawbacks. In the context of tiered-memory systems, if a node's threshold is too low, fast nodes may not reclaim aggressively enough, and there will be no space available to promote hot pages from the slow-tier nodes. 

#### Optimizing reclaim and demotion

A recent [patch set](<https://lwn.net/ml/linux-mm/20220221084529.1052339-1-ying.huang@intel.com/>) by Huang Ying highlights and addresses this problem. The premise behind this work is that the working-set size of workloads on systems with multiple memory types will, in the common case, exceed the total amount of fast DRAM in the system. This makes sense. If a system wasn't overcommitting DRAM, there would be no need to use other memory types in the first place. 

The implication of this insight is that, on tiered-memory systems, pages will be constantly moved between fast and slow memory nodes as they're accessed by the application. If the fast nodes are near capacity, the kernel won't be able to promote globally hot pages into those nodes during rebalancing; resulting in higher-than-necessary access latencies due to hot pages residing on slow-tier nodes. The trick is, therefore, ensuring that sufficient pages are reclaimed from fast nodes such that, in addition to making space for future allocations, there is also enough room in the fast nodes to promote hot pages from slower nodes. 

Ying's patch set addresses this need by introducing a new `WMARK_PROMO` watermark that is larger than the (previously highest) `WMARK_HIGH` watermark. When a page is unable to be migrated to a faster node due to memory pressure, `kswapd` is woken up to reclaim memory up to the new `WMARK_PROMO` threshold. This slightly more aggressive reclaim strategy better ensures that there is sufficient space for hot pages to be promoted from the slow memory nodes to the fast memory nodes, and thus better accommodates the working sets that are common on tiered-memory systems. 

#### The controversy of statically sized watermarks

While adding the `WMARK_PROMO` watermark improves the chances that there will be sufficient space on fast nodes to promote hot pages from slower nodes, one has to wonder whether the general notion of static watermarks should be revisited. Consider that, even if the chosen watermark threshold is sufficiently high to ensure that pages may be promoted to a fast node, a threshold that is higher than necessary will leave DRAM unused, and the application's performance will be negatively impacted. The fact that a new watermark was required in the first place is indicative of the nature of the problem, which is largely dependent on both the characteristics of the system itself, and the workloads it's running. 

The drawbacks of using a static watermark were discussed in reviews of the patch. For example, with regard to an [earlier version](<https://lwn.net/ml/linux-kernel/20211116013522.140575-3-ying.huang@intel.com/>) of Ying's patch, which hard-coded the number of additional pages required during reclaim to be 10MB larger than `WMARK_HIGH`, Zi Yan [questioned](<https://lwn.net/ml/linux-kernel/C9EB54B4-AA86-4473-AE04-95E3CC6F6E94@nvidia.com/>) whether such a value made sense: 

> Why 10MB? Is 10MB big enough to avoid creating memory pressure on fast memory? This number seems pretty ad-hoc and may only work well on your test machine. 

Ying [acknowledged](<https://lwn.net/ml/linux-kernel/87bl2gsnrd.fsf@yhuang6-desk2.ccr.corp.intel.com/>) that the 10MB value was hard to justify and that there was room for improvement beyond the current implementation. The threshold was subsequently changed into the separate `WMARK_PROMO` watermark, based on a [suggestion](<https://lwn.net/ml/linux-kernel/YgFbQFlZ+2rvS9Mm@cmpxchg.org/>) by Johannes Weiner, who also pointed out that another option was to have promotions dynamically boost the `kswapd` watermarks on demand. This would avoid the problem of DRAM being under-utilized, though of course it would also come at the cost of increased complexity. 

There is certainly nothing wrong with incremental improvements, nor with sticking with a simple approach until more complexity is required. It will be interesting to see, however, whether the kernel will eventually require a more dynamic and flexible framework for expressing decisions regarding reclaim and page migration. 

#### Avoiding page ping-pong

In addition to requiring fast nodes to have sufficient space for promoted pages, there is another problem that is unique to tiered-memory systems. In conventional NUMA setups, application working sets are typically sized to fit into one or more nodes. Once the application has reached a steady state and most or all of the pages are correctly located on the nodes where they're locally accessed, migration should taper off. Applications on tiered memory systems do not behave this way, though, since their working sets may not fit into the NUMA nodes they are running on. Rather than reaching a steady state, pages are instead continuously ping-ponged back and forth between slow and fast NUMA nodes as they're accessed by applications. 

This is a related, but different problem than the one solved by Ying's patchset. The new watermark ensures that there's sufficient space on a fast node for hot pages to be promoted from a slow node, but doesn't prevent pages from being continuously and aggressively migrated between cold and fast nodes. If the overhead of performing the migration exceeds the performance gained from the improved access latencies of having a page on a local DRAM node, then the rate of page migration clearly needs to be adjusted. There have been multiple proposals for how to solve this issue. 

One [proposal](<https://lwn.net/ml/linux-kernel/20211116013522.140575-5-ying.huang@intel.com/>) by Ying involves recording the time that passes between a page in slow memory being unmapped to create a NUMA hint fault and when that fault is actually observed on a memory access. The shorter the time that the page was unmapped, the more likely it is that a page is actually hot. That time is compared to some threshold (tunable by a system administrator), and the page is only promoted if the elapsed time is within that threshold. 

While time-since-access feels like a natural way to quantify page hotness, the approach is also quite complex, and requires adding a lot of new code. The question of what the threshold defining a page to be considered "hot" should be is also unclear; tuning by a system administrator may be required. A [follow-on patch](<https://lwn.net/ml/linux-kernel/20211116013522.140575-7-ying.huang@intel.com/>) proposed a method to dynamically tune the threshold based on the volume of migrations, but it, too, is quite complex. 

An alternative approach was proposed in a [patch set](<https://lwn.net/ml/linux-kernel/80a0cfd545a01ce0470a727cb961a5e0a1532d48.1637778851.git.hasanalmaruf@fb.com/>) by Hasan Al Maruf. When a page is demoted, it is removed from the active LRU list and placed onto the inactive LRU list. Al Maruf's patch updates the NUMA hint-fault handler to check whether a page is in this inactive state and, if so, move it to the active state and defer promotion until a subsequent fault. If the page is once again accessed, it will be observed as present on the active LRU, and the promotion will occur. The advantage of this solution is that it uses an existing mechanism in the kernel for tracking page hotness. As memory pressure increases and more pages are reclaimed, more pages are moved to the inactive LRU list, thus causing page promotion to be throttled proportionally. 

A consensus has not yet been reached on which solution will be chosen, though Al Maruf's patch set will likely be accepted thanks to its simplicity and its use of existing mechanisms for tracking a page's hotness. While the solution is not expected to be controversial, there is always the [Linux Storage, Filesystems, and Memory-Management Summit](<https://events.linuxfoundation.org/lsfmm/>) around the corner, where developers can discuss the merits of each approach in person.
