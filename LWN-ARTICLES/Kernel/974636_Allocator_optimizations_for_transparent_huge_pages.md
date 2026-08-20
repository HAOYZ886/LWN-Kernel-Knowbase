---
title: Allocator optimizations for transparent huge pages
url: https://lwn.net/Articles/974636/
date: "May 24, 2024"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet May 24, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 24, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The original Linux kernel, posted in 1991, ran on a system with a 4KB page size. Over 30 years later, most of us are still running on systems with 4KB pages, even though the amount of installed memory has grown by a few orders of magnitude. It is generally accepted that using large page sizes results in better performance for most applications, but allocating larger pages is often difficult. During a memory-management session at the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Yu Zhao presented his ideas on improving the allocation of huge pages in the kernel. 

It is worth noting that this session was focused on a patch set that was [examined here in March](<https://lwn.net/Articles/964239/>). Zhao did not go deeply into the details of how his improved allocator works in the session; reading that article now could provide some useful background. 

[![\[Yu Zhao\]](https://static.lwn.net/images/conf/2024/lsfmm/YuZhao-sm.png)](<https://lwn.net/Articles/974642/>) Zhao started by saying that "some CPU vendor" is planning to drop 4KB pages entirely within the next decade. MacOS on Arm systems uses 16KB pages now, and Google is experimenting with 16KB pages on Android. He made the proposition that 4KB pages are suboptimal for modern user space, but the problem remains that some architectures do not support any other size. Additionally, changing the base-page size is an ABI break that can cause problems for some applications. 

Thus, he said, "a forward-looking operating system would offer the opportunity to favor larger logical pages". That system would treat 4KB pages as a legacy feature, but would not require a larger base-page size or break existing ABIs. Favoring huge pages over 4KB pages, he said, brings better performance and lower metadata overhead; that will be even more true once [the plan to switch to memory descriptors](<https://lwn.net/Articles/973565/>) becomes reality. 

The problem is that the ability to allocate 4KB pages fragments system memory; defragmentation imposes a cost, and may be impossible. That results in an economy where 4KB pages are cheap, and huge pages are expensive. The cornerstone of his THP allocation optimization (TAO) proposal is turning that situation around, making huge pages cheap, and 4KB pages expensive. 

The ability to assemble huge pages depends partly on the ability to move small pages out of the way. The kernel provides allocation-time hints like `__GFP_MOVABLE` now so that allocations that can (hopefully) be moved are located together. Unmovable allocations are a problem, though; they block assembly of huge pages, and their lifetime is not predictable. There is a research project at Google (called "Tetris") that is aimed at determining that lifetime, using statistical sampling and estimation, with the goal of grouping unmovable allocations by lifetime. 

Low-priority tasks, Zhao said, can fragment memory, impacting the performance of higher-priority tasks. It would be nice to be able to isolate those low-priority tasks, but that needs support from the memory controller and, perhaps, cooperation from user space. But another key component (and a key part of the TAO patches) is memory partitioning. Fragmentation can be irreversible, he said, so it is best to avoid it by isolating the smaller allocations in a separate memory partition. A well-chosen partitioning scheme, he said, can readily provide huge pages while applying a higher level of memory pressure to applications that are making a lot of small allocations. 

Shakeel Butt asked whether the zone for 4KB allocations would be limited to movable allocations or not. Zhao replied that it depends on the fallback order that is chosen. If, as he suggests, the kernel attempts to allocate compound (huge) pages before falling back to 4KB pages, then there can be unmovable objects in the 4KB zone. 

Setting up partitions raises the issue of sizing. Zhao's proposal sets global minimum and maximum limits on the size of the huge-page partition, but that is only part of the problem. Low-priority tasks could still hog the huge pages, so there will have to be a limit, enforced by a control group, on use of the huge-page partition. It will be possible to resize the partitions based on the workload, but that requires memory hotplugging. Shrinking the huge-page partition should be guaranteed, since those allocations are all movable; moving in the other direction would be a best-effort affair. 

A participant asked where the line would be drawn between good (large) and bad (small) allocations. Zhao answered that it depends on the system. For many, it would be the CPU's continuous-PTE size (often 16KB or 64KB); on servers it would be the PMD size, which (at 2MB typically) is rather larger. There was some inconclusive discussion on what the best size to use might be. 

Zhao continued, saying that automatic resizing of the partitions will be needed, based on their relative memory pressure. The 4KB partition would be allowed to have a higher pressure as a way of fighting fragmentation. He suggested that memory pressure in the 4KB partition could invoke the out-of-memory (OOM) killer, even if the huge-page partition is not having problems. There are a number of platforms that use OOM kills as part of their ordinary operation; Android, ChromeOS, and cloud providers (to manage batch jobs) are all examples, so bringing in the OOM killer is not necessarily a bad thing. The alternative, he said, would be to watch the huge-page partition fade away due to fragmentation over time. 

Zhao presented some plots showing that systems running with the TAO patches benefit from improvements in both huge-page allocation rates and web-browser responsiveness. 

David Hildenbrand asked whether the partition resizing could be done using the memory-management subsystem's page-block abstraction rather than hotplugging; Vlastimil Babka replied that page blocks do not have separate free lists, so they cannot be used to direct allocations in the same way. Hildenbrand suggested that perhaps extending page blocks might be the right approach; on big systems, he said, nobody is able to cope with the complexity of hotplugging. He would not be able to convince RHEL users to use the TAO feature. Configuring phones, which run a single workload, is easy; servers are rather harder. 

Johannes Weiner pointed out that he had posted [a patch set](<https://lwn.net/ml/linux-mm/20230418191313.268131-1-hannes@cmpxchg.org/>) for reliable huge-page allocation last year. Reviewers asked him to split the work apart; some of it is staged to go into the 6.10 release. He was able to get a success rate of 99% for 2MB huge-page allocations; that is good enough, he said. Larger allocations are only of interest to a small group of users. 

Zhao concluded the session by speaking briefly about the longer-term goals of his work. They include using TAO to provide huge pages to back up hugetlbfs, and the ability to reliably allocate 1GB huge pages.
