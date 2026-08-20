---
title: The future of memory tiering
url: https://lwn.net/Articles/931421/
date: "May 12, 2023"
category: "Compute Express Link CXL; Memory management-Tiered-memory systems"
author: "By Jonathan Corbet May 12, 2023 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 12, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Memory tiering is the practice of dividing physical memory into separate levels according to its performance characteristics, then allocating that memory in a (hopefully) optimal manner for the workload the system is running. The subject came up repeatedly during the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>). One session, led by David Rientjes, focused directly on tiering and how it might be better supported by the Linux kernel. 

Tiering was often mentioned in the context of [CXL](<https://www.computeexpresslink.org/>) memory but, Rientjes began, it is not just about CXL. Instead, tiering presents a number of "generic use cases for hardware innovation". There are a lot of ways of thinking about tiering and what is covered by the term. The management of NUMA systems, where some memory is closer to a given CPU (and thus faster to access) than the rest, is a classic example. Swapping could be seen as a form of tiering, as can non-volatile memory or high-bandwidth memory. And, of course, mechanisms like CXL memory expansion and memory pooling. It is, he said, leading to "a golden age of optimized page placement". 

[![\[David Rientjes\]](https://static.lwn.net/images/conf/2023/lsfmm/DavidRientjes-sm.png)](<https://lwn.net/Articles/931524/>) The discussion briefly digressed into whether swapping really qualifies as tiering. In the end, the consensus seemed to be that, to be a memory tier, a location must be byte-addressable by the CPU. So swapping is off the list. 

Michal Hocko said that there are two dimensions to the tiering problem. One is the user-space APIs to be provided by the kernel; somehow user space has to be given the control it needs over memory placement. The relevant question here is whether the existing NUMA API is sufficient, or whether something else is needed. The other aspect of the problem, though, is the kernel implementation, which should handle tiering well enough that user space does not actually need to care about it most of the time. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Rientjes responded that the NUMA API has been a part of the kernel for around 20 years. Whether it is suitable for the tiering use case depends on the answers to a number of questions, including whether it can properly describe and control all of the types of tiering that are coming. Slower expansion memory is the case that is cited most often, but there are others, including memory stored on coprocessors, network interfaces, and GPUs. He wondered what kinds of incremental changes to the NUMA API would be needed; the one-dimensional concept of NUMA distance may not be enough to properly describe the differences between tiers. The group should also, he said, consider what the minimal level of kernel support should be, and which optimizations should be left to user space. 

One problem, Dan Williams said, is that vendors (and their devices) often lie to the kernel about their capabilities. Getting to the truth of the matter is not a problem that can just be punted to user space. There need to be ways for user space to indicate its intent, which can then be translated by the kernel into actual placement decisions. 

Matthew Wilcox said that systems will have multiple levels of tiering; the kernel will have to decide how to move pages up and down through those tiers. Specifically, should movement be done one step at a time, or might it be better to just relegate memory directly to the slowest tier (or to swap) if it is known not to be needed for a while? And if multi-tier movement is the right thing to do, how does the kernel figure out when it is warranted? After a bit of inconclusive discussion, Hocko repeated that, while it would be nice to push decisions like that to user space, the kernel has the responsibility to do the right thing as much as possible. 

Rientjes had a number of other questions to discuss, but the time allotted to the session was running out. The biggest problem for memory tiering still appears to be page promotion; it is not particularly hard to tell when pages are not in use and should be moved to slower memory, but it is rather more difficult to determine when a page has become hot and should be migrated to faster storage. There are a number of options being explored by CPU vendors to help in this area; the kernel is going to have to find a generic way to support these forthcoming architecture-specific features. 

A few other questions had to be skipped over. One of these was what the interface for the configuration of memory devices as tiered nodes should look like. User space will want to influence tiering policies, but that interface has yet to be designed as well. Probably some sort of integration with control groups will be necessary. The list of questions went on from there, but they will have to be discussed some other time.
