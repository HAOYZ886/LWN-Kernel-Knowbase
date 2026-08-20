---
title: "Preparing DAMON for future memory-management problems"
url: https://lwn.net/Articles/1016525/
date: "April 10, 2025"
category: "Memory management-DAMON"
author: "By Jonathan Corbet April 10, 2025 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 10, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025>)

The [Data Access MONitor (DAMON)](<https://docs.kernel.org/admin-guide/mm/damon/>) subsystem provides access to detailed memory-management statistics, along with a set of tools for implementing policies based on those statistics. An update on DAMON by its primary author, SeongJae Park, has been a fixture of the Linux Storage, Filesystem, Memory-Management, and BPF Summit for some years. The 2025 Summit was no exception; Park led two sessions on recent and future DAMON developments, and how DAMON might evolve to facilitate a more access-aware memory-management subsystem in the future. 

#### A status update

The first session was dedicated to updating participants on what is happening with DAMON; Park started by thanking all of the members of the development community who have contributed changes. DAMON, he reminded the group, provides access-pattern information for both virtual and physical address spaces, and allows the specification of access-aware operations to be performed. Thus, for example, an administrator can set up a policy to reclaim all pages that have been idle for at least two minutes. DAMON is increasingly used for this sort of proactive reclaim by large providers like Amazon's AWS, and in various memory-tiering settings. 

[![\[SeongJae Park\]](https://static.lwn.net/images/conf/2025/lsfmm/SeongJaePark-sm.png)](<https://lwn.net/Articles/1016707/>) Major changes to DAMON since [the 2024 update](<https://lwn.net/Articles/973702/>) include [a new tuning guide](<https://docs.kernel.org/next/mm/damon/design.html#damon-design-monitoring-params-tuning-guide>). Page-level monitoring has been enhanced to expose information about huge pages that can be used in policy filters. Shakeel Butt asked whether this filter would select specific huge pages, or those within a given region; Park said that the filter can be applied to any address range. DAMON can provide more detailed information, including what proportion of a given region has passed a filter. There are filters for huge-page size and whether pages are on the active least-recently-used (LRU) list. 

DAMON now has some automatic tuning for monitoring intervals. Getting the interval right is important; if it is too short, all pages will look cold, but making it too long makes all pages look hot. The default interval (100ms) is ""not magic"", he said. DAMON users have all been doing tuning in their own ways, duplicating a lot of effort. The tuning knobs were finally documented, in the above-mentioned guide, in 6.14, which is a start; documentation is good, he said, but doing the right thing automatically is better. 

The important question is: how many access events should each DAMON snapshot capture? The answer is the "access event ratio" — the number of observed events divided by the number that could have been observed. The automatic tuner runs a feedback loop aiming for a specified ratio; in the real world, it tends to converge near 370ms for a normal load, but four or five seconds on a light load. The result is good snapshots with lower overhead. Given how well the feature seems to work, Park wondered whether it should be enabled all the time. 

Michal Hocko said that DAMON is not enabled in the SUSE distributions because nobody has asked for it. So, while he has no problem with _enabling_ this tuning by default, he does not like the idea of it being _active_ by default. 

In the 2024 discussion on using DAMON for memory tiering, a simple algorithm was proposed: 

  * On each NUMA node lacking a CPU, look at how that node sits in the hierarchy: 

    * If the node has lower nodes (hosting slower memory), then demote cold pages to those lower nodes, aiming for a minimum free-memory threshold. 
    * If the node has upper (faster) nodes, promote the hot pages, aiming for a minimum used-memory threshold on the upper node.  

The core patches enabling this algorithm were merged for the 6.11 kernel release. A DAMON module has been implemented to manage the tiering, and [an RFC posting](<https://lwn.net/ml/all/20250320053937.57734-1-sj@kernel.org/>) has been made. When asked whether this module could make use of access information not collected by DAMON itself, Park answered that there is no such capability yet, but agreed that it would be useful. 

In Park's testing system, use of the DAMON tiering module improved performance by 4.4%, but the existing NUMA-balancing-based tiering degraded instead. So there is a need for more investigation, but he concluded that, sometimes at least, DAMON can improve performance for tiering workloads. 

Park ended the session with a brief discussion of where DAMON is going next. He wants to merge his work on generating NUMA utilization and free-space goal metrics, and implement a ""just works"" tiering module, suitable for upstream, that is able to identify a system's tiers automatically. Thereafter, this module could be extended to more general heterogeneous memory-management tasks. Taking CPU load and available memory bandwidth into account when performing migration was the last item on the list. 

#### Future requirements

DAMON remained on the agenda for the next session, though, which was focused on requirements in a future where the memory-management system has more visibility into access patterns. There are a number of proposals circulating for ways to acquire that data, Park said, including working-set reporting, hotness monitoring from [CXL](<https://en.wikipedia.org/wiki/Compute_Express_Link>) controllers, accessed-bit scanning (as was [discussed](<https://lwn.net/Articles/1016519/>) earlier in the day), data from AMD's Instruction Based Sampling feature, data from the [multi-generational LRU](<https://lwn.net/Articles/856931/>), and more. The time is clearly coming when it will be necessary to handle multiple sources of access information to better manage memory. With that information, it should be possible to provide better control to administrators, automatically unblock a primary workload's progress, or achieve better fairness between tasks. 

DAMON provides an "operations set" layer called DAMOS that handles the implementation of fundamental memory-management operations. That includes managing sources of information, which is then fed into the DAMON core and distilled into region-specific info. DAMOS operates under quotas that bound its resource usage; they can be specified by the administrator or tuned automatically. There are also filters that can narrow its attention to specific pages. 

In the future, Park said, he would like to reduce the overhead of DAMON while improving its accuracy. If it can be made more lightweight, ""everything will be solved"". New data sources will help toward that goal. He plans to add an optimized API for access reporting; specifically, there will be a new function, `damon_report_access()`, to provide memory-access information to the kernel. It will take the ID of the accessing process, the address of interest, the number of accesses, and the NUMA node from which the access is made. The plan is for this function to be callable from any context. 

There may also be a new function, `damon_add_folios()`, to indicate that specific folios are the highest-priority target for memory-management operations. It can be used to identify folios for promotion or demotion, for example. 

He will be posting `damon_report_access()` soon, he said, so that DAMON can take advantage of access information from sources beyond its own scanning. The need for `damon_add_folios()` is less clear at the moment; he mostly just imagines that there might be a use for it. Park would like to hear from people with use cases for this functionality. 

Jonathan Cameron said that the fact that a hardware-based access-reporting system does _not_ report on a specific range is also significant; he suggested adding an interface for pages that are known not to have been accessed. He also said that, as finer-grained access data comes from more sources, the regions that DAMON considers will be broken up more finely in response. That will lead to denser data overall, and more for DAMON to track; he wondered if Park had thoughts on reducing the resulting tracking overhead. Park answered that DAMON already lets users set limits on the number of regions, and that this keeps DAMON in check now; it should work in the future as well. 

The last question came from John Groves, who wondered if the access-reporting interface should be tied to virtual memory areas (VMAs). A mapped file will be represented by a single VMA, he said, even if it has a lot of shared users, so that could be a logical way to manage it. Park answered that a VMA-based interface could maybe make sense, but he would need to hear more about the uses cases for it.
