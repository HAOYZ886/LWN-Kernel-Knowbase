---
title: An update and future plans for DAMON
url: https://lwn.net/Articles/973702/
date: "May 17, 2024"
category: "Memory management-DAMON"
author: "By Jonathan Corbet May 17, 2024 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 17, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The [DAMON](<https://docs.kernel.org/mm/damon/index.html>) subsystem was the subject of the first session in the memory-management track at the [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). DAMON maintainer SeongJae Park introduced the data-access monitoring framework, which can generate snapshots of how memory is accessed, enabling the detection of hot and cold regions of memory in both the virtual and physical address spaces. The session covered recent changes and future plans for this tool. 

While DAMON can acquire memory-usage information, DAMOS extends DAMON by enabling the specification of policies to take action on that information. It can, for example, be instructed to force out any page of memory that has not been accessed in the last five seconds. Recent work on DAMOS includes the addition of a quota feature to control how aggressively it works; it can be used to limit the amount of memory processed in a given time period. There is also a new filter mechanism to better focus its efforts; for example, DAMOS can be directed at specific NUMA nodes, or to only work on file-backed pages. 

[![\[SeongJae Park\]](https://static.lwn.net/images/conf/2024/lsfmm/SeongJaePark-sm.png)](<https://lwn.net/Articles/973772/>) DAMOS is seeing increased use, Park said. A number of products are using it now for proactive reclaim, and there is interest in using it for [Compute Express Link (CXL)](<https://lwn.net/Articles/894598/>) memory management. DAMOS has also been picked up by researchers, leading to some 20 citations in the literature. 

At [a 2023 LSFMM+BPF session](<https://lwn.net/Articles/931769/>), Park was told that better documentation for DAMON would be appreciated; that documentation has since been written and merged. That session also concluded that keeping the DAMON user-space tools in the kernel tree would not be a good idea. Part of the motivation for raising that idea had been to generate better test coverage. Park is now working on adding that to kselftest instead. 

Another improvement is pseudo moving-sum-based fast snapshots. By default, DAMON produces snapshots over a period of 100ms. In 6.7, it gained the ability to create "reasonable snapshots" over shorter sampling intervals, 5ms by default. That is useful when the user wants to aggregate data over longer intervals, but would like to be able to get shorter-term data as well. 

There are some new filter types. Aggregation can now be filtered on address ranges, and narrowed to NUMA nodes, memory zones, or virtual-memory areas. DAMON can also filter on the "page is young" flag, which can be used to double-check the status of a page before acting on it. The biggest change, though, is "aim-ordering, feedback-driven, aggressive auto-tuning". It allows the DAMOS quota to be automatically adjusted with a feedback loop. The user can provide the quota value, based on a parameter such as workload latency, or the kernel can drive it using existing system metrics, such as targeting a given [pressure-stall](<https://lwn.net/Articles/759781/>) rate. 

#### What's next

Looking to the future, the first objective is control of tiered-memory management with automatic tuning; this is an area that is being explored now. The initial objective will be two-tier promotion and demotion; some patches are available now. The algorithm, roughly, is proposed to eventually work like this: 

  * If a node has a lower (slower) node available to it, then demote cold pages to that lower node, keeping the amount of free memory above a minimum threshold. 
  * If the node has an upper (faster) node, then push hot pages up to the upper node, trying to keep the utilization rate on that node high. 

The objective here is to maximize the utilization of memory on the faster nodes, while keeping pages that are accessed less frequently in slower memory. The algorithm aims for a slow but continuous movement of pages between nodes, and will be extendable to systems with more memory tiers. 

Another objective is ["access/contiguity-aware memory auto scaling"](<https://lwn.net/ml/linux-kernel/20240512193657.79298-1-sj@kernel.org/>) or ACMA. The model here is that the user will specify the minimum and maximum memory requirements for their workload; a service provider will then run the workload somewhere, aiming for both good performance and low cost. Optimizing this scenario in current kernels requires the orchestration by the provider of four kernel features: memory overcommit, reporting more reclaimable pages with DAMON, periodic compaction, and memory hotplug to set hard limits and to minimize the `page` structure overhead. 

Systems using these techniques have been working well in real-world deployments for years, Park said. But, he added, it is also a rather complex solution. Relying on memory hotplug is both slow and prone to failure — there are many ways to block the hot-removal of memory. System-level memory compaction is wasteful, especially in the absence of access information. Users can access pages at any time, thwarting the system's efforts to better organize memory. As a result, non-collaborative control of guests is difficult or impossible. 

Park proposed an alternative for allocation of memory to guests based on two core actions. `damos_alloc` will allocate a memory region with a minimum level of contiguity, then inform the user about that allocation; `damos_free` returns memory to the system, also maintaining minimum levels of contiguity. These actions are driven by the system's current pressure-stall level. Memory is allocated to keep the stall level below an acceptable maximum, while freeing happens to keep that level above a minimum threshold. Since notifications are provided for memory changes, collaborative guests can react accordingly; ballooning can be used to control non-collaborative guests. 

The objective is to limit the complexity involved in making such a system work; there are just three parameters to adjust. Since ACMA scales memory in 2MB chunks, it maintains the contiguity of memory on the host, even under high memory pressure. This system could also be extended to support the [contiguous memory allocator](<https://lwn.net/Articles/486301/>) or for power management by powering down memory banks when they are not needed. 

Michal Hocko pointed out that the kernel should be providing mechanisms rather than policy, and asked how user space would control this feature. Park answered that control is currently managed through the DAMON sysfs interface, but the plan is to create simpler modules with fewer knobs to adjust. Hocko said that he was concerned about creating long-term API issues; developers are still trying to figure out what the best interfaces should be for the control of memory tiering, and it is important to be careful about which interfaces we commit to. "Sysfs is terrible", he continued; it allows the addition of too many interfaces without sufficient review. There needs to be more consideration of the API before this work can be merged. 

Dan Williams asked whether there was a path to migrate DAMON-based features to more formal kernel interfaces. DAMON is a good way to do "science experiments", he said, but perhaps there should be a promotion path into the core kernel for the experiments that succeed. David Hildenbrand expressed worries about interference with the core memory-management code, and said that it was important that DAMON doesn't start taking on too much work. As the session ran out of time, Park said that he is trying to keep DAMON simple and to avoid that kind of interference.
