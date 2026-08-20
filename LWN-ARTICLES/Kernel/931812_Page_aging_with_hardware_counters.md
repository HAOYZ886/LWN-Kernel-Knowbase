---
title: Page aging with hardware counters
url: https://lwn.net/Articles/931812/
date: "May 18, 2023"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet May 18, 2023 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 18, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The memory-management subsystem has the unenviable task of trying to predict which pages of memory will be needed in the near future. Since predictions tend to be difficult, the code relies heavily on the heuristic that memory used in the recent past is likely to be used again in the near future. However, even knowing which memory has been recently used can be a challenge. At the 2023 [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Aneesh Kumar and Wei Xu, both presenting remotely, discussed some ways to use the increasingly capable hardware counters that are provided by current and upcoming CPUs. 

[![\[Aneesh Kumar\]](https://static.lwn.net/images/conf/2023/lsfmm/AneeshKumar-sm.png)](<https://lwn.net/Articles/931916/>) Kumar started by talking about how these counters, which can count page accesses, might be used to improve the [multi-generational LRU](<https://lwn.net/Articles/856931/>); he has recently posted [a patch series](<https://lwn.net/ml/linux-mm/20230402104240.1734931-1-aneesh.kumar@linux.ibm.com/>) implementing this functionality. It uses page-access counters to help with the sorting of pages into generations. Counters help, but do not entirely solve the problem, he said; most architectures can produce access counts at this point, but those are absolute numbers. Page sorting requires a sense of _relative_ activity — which pages are seeing more activity than others now? The code works by looking at the counts for some pages in the oldest and youngest generations, using them to estimate what the minimum and maximum numbers would be. Those values are then used to classify other pages. He said that it might be feasible to use [k-means clustering](<https://en.wikipedia.org/wiki/K-means_clustering>) to classify pages instead. 

The advantage of using the counters is that the kernel can skip the work of walking through a process's page tables to find the pages that have been accessed recently. It also eliminates the need to store generation information in the page flags, which are a tightly limited resource. The results from benchmarking this code were somewhat inconclusive, though; some workloads regressed slightly, while others improved a little. Optimizing the multi-generational LRU, he said, is a tricky task. Profiling the kernel showed that the use of hardware counters reduced the time spent checking per-page "accessed" bits, but added time spent querying the counters. In summary, he said, it is still not entirely clear whether this feature is worth adding or not. 

An entirely different use case is measuring the utilization of transparent huge pages. These pages, which are often made up of 512 4KB "base" pages, have a single bit tracking references to the whole thing. If a single base page within a huge page is active, it make the whole huge page look busy, even though the other 511 base pages might be unused. Hardware counters exist for each base page, though, so they should be able to identify hot and cold pages hidden within a huge page. 

The approach he is working on changes the `khugepaged` thread, which "collapses" base pages into huge pages behind the scenes, with the result that it only creates a huge page if all of the base pages that it contains have approximately equal access frequencies. The reclaim process can look at access patterns for the base pages within a huge page and break that huge page apart if it is sparsely used. Benchmarking this work was tricky, since it is hard to find workloads that show this type of access pattern. With a special-purpose microbenchark, though, Kumar was able to demonstrate that sparsely-used huge pages can be split, freeing little-used memory for other uses. 

Another potential use for hardware counters is in page promotion, which relies on being able to detect heavily used pages that should be promoted to faster storage. Using hardware counters, the kernel can compare the relative hotness of pages across NUMA nodes, which is not easily done now. Kumar has been unable to test this idea, though, since he lacks the hardware that it would apply to. 

Assuming that there is a place for hardware counters, developers would need to find the best way to integrate them. One option, he said, was to add support to [DAMON](<https://lwn.net/Articles/931769/>), but that approach is hard to evaluate. One benchmark he ran showed a 12% performance improvement, but it also showed a lot of variance. 

[![\[Wei Xu\]](https://static.lwn.net/images/conf/2023/lsfmm/WeiXu-sm.png)](<https://lwn.net/Articles/931917/>) Xu took over at this point to return to the question of page promotion, which he described as a key challenge for memory tiering. There are currently a number of ways of identifying hot pages (which are candidates for promotion) in the kernel, he said, including page faults, accessed-bit scans, hardware counters, and more. It would be good to somehow unify the kernel's approach to hardware-assisted page promotion. The best approach there, he said, would be to create an abstraction layer around page promotion that could use a number of back-end modules to acquire information on page usage. 

He has implemented a user-space promotion daemon that used a combination of access bits and precise event-based sampling (PEBS) data, with events streamed to user space by way of a BPF interface. In combination with a custom system call to request the migration of physical pages, he said, it worked "well enough". 

He wondered how this kind of functionality might be brought into the kernel. One possibility would be to extend the [autonuma](<https://lwn.net/Articles/488709/>) mechanism to use hardware counters but, he said, that is not a great fit. Autonuma is based on virtual memory areas, while the counters are tied to page-frame numbers. A better idea, he thought, was to extend the multi-generational LRU to make use of this information. It could be augmented with a per-node page-promotion thread that works like kswapd, but in the opposition direction. 

Kumar asked the gathered developers how they thought the incorporation of hardware counters should proceed. Dan Williams said that user space might also want to use the system's performance counters; if the kernel grabs them, those applications could break. Kumar answered that this would not be a problem on architectures, like PowerPC, that have dedicated counters for this purpose. Williams suggested implementing this functionality for such hardware first. Xu added that he had used PEBS for his work because it was the only thing available, but that dedicated counters are a better solution and he hoped other vendors would start to provide them. 

DAMON developer SeongJae Park expressed his thanks for the DAMON integration, which is something that he had been wanting to do himself; he encouraged the sending of patches. Kumar said that the patches would be sent, but remarked that DAMON is difficult to use for generic workloads; Park answered that he is working on automatic tuning to address that problem. 

The session closed with a suggestion from Williams that proving the value of hardware counters in DAMON would be a good first step. After that, if it seems worthwhile, support for these counters could be moved into the core memory-management code.
