---
title: HMM and CDM
url: https://lwn.net/Articles/717601/
date: "March 22, 2017"
category: "Memory management-Heterogeneous memory management"
author: "By Jonathan Corbet March 22, 2017 LSFMM 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
March 22, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

The first topic in the memory-management track of the 2017 Linux Storage, Filesystem, and Memory-Management Summit continued [a discussion](<https://lwn.net/Articles/717614/>) begun during the preceding plenary session, which had introduced the heterogeneous memory management (HMM) issue and updated the group on the status of those patches. In this session, Jérôme Glisse focused the discussion on what needs to be done to move this work forward. Balbir Singh then followed up with a different approach to the HMM problem where more hardware support is available. 

#### Pushing HMM forward

The HMM discussion started with a question from the group: what features does a device like a GPU need to be able to support HMM? The answer is that it needs some sort of a page-table structure that can be used to set the access permissions on each page of memory. That enables, for example, execute permissions to be set on either the CPU or the GPU (but not on both), depending on which kind of code is found in the relevant pages. HMM also needs to be able to prevent simultaneous writes by both processors, so the GPU needs to be able to handle faults. 

Dave Hansen asked whether more was needed than an I/O memory-management unit (IOMMU) could provide. Glisse responded that the IOMMU is there primarily to protect the system from I/O devices, which is a different use case. Mel Gorman added that HMM needs to be able to trap write faults on a specific page and provide different protections on each side — things an IOMMU cannot do. 

There is work underway to use the [KSM](<https://lwn.net/Articles/330589/>) mechanism to do write protection; those patches will be posted soon. KSM allows the same page to be mapped into multiple address spaces, a feature [![\[Jérôme Glisse\]](https://static.lwn.net/images/conf/2017/lsfmm/JeromeGlisse-sm.jpg)](<https://lwn.net/Articles/717624/>) which will be useful, especially on systems with multiple GPUs, all of which need access to the same data. 

Andrea Arcangeli started a discussion on the handling of write faults. Normally such faults on shared pages lead to copy-on-write (COW) operations, but the situation here is different; the response in the HMM setting is to ensure that the writing side has exclusive access to the memory in question. Gorman raised some worries about the semantics of writes after `fork()` calls by processes using HMM. `fork()` works by marking writable pages for COW, but it's not clear what should happen if the pages are COW-mapped into both the parent and child; a write fault could end up giving ownership of the page to one process or the other in a timing-dependent (i.e. racy) manner. 

To avoid this eventuality, Gorman suggested that all memory used with HMM should be marked as `MADV_DONTFORK` using the `madvise()` system call; that would cause that memory to not be made available to the child of a `fork()` at all. Indeed, he said that it should be mandatory. He relented a bit, though, after it was explained that all HMM memory is pulled into the parent process at `fork()` time, with none left in the GPU. He was willing to accept the situation as long as it is clear that the HMM memory is associated with the parent and is not visible to the child. 

With that resolved, Gorman asked if there were any remaining obstacles to merging. Hansen mentioned that HMM will not work with systems that already have the maximum amount of memory installed; there simply is no physical address space for the GPU memory. Gorman replied that this problem will indeed come up in practice and users will be burned by it, but that it is a limitation of the hardware and is not a reason to block the merging of the HMM patches. 

Dan Williams expressed a concern that the HMM patches place GPU memory into the `ZONE_DEVICE` zone, which is also used for persistent memory. The two uses are distinct and can get along, but the code around `ZONE_DEVICE` becomes that much easier to break if a developer making a change doesn't understand all of the users. Gorman suggested that Williams should do a detailed review of the HMM code from a `ZONE_DEVICE` perspective; the long-term maintainability of this code is a fundamental issue, he said, and needs to be considered carefully. Johannes Weiner suggested, in jest, that `ZONE_HIGHMEM` could be used instead, to which Gorman told him to "go home". 

The final concern is the lack of any drivers for the HMM code; if it is merged in its current form, it will be dead code with no users. There is, it seems, some hope that a Nouveau-based driver for NVIDIA GPUs will be available by the time the 4.12 merge window opens. Gorman suggested to Andrew Morton that the HMM code could be kept in the -mm tree until at least one driver becomes available, but Morton asked whether it would really be a problem for the code to go upstream as-is. What he would most like, instead, is a solid explanation of what the code is for so he can justify it to Linus Torvalds when the time comes. 

The end result is that HMM has a few hurdles to get over still, but its path into the mainline is beginning to look a little more clear. 

#### Coherent device memory nodes

Singh then stepped forward to describe the IBM view of HMM; for IBM, the problem has been mostly solved in hardware. On suitably equipped systems, the device memory shows up as if it were on its own NUMA node that happens to lack a CPU. That memory is entirely cache-coherent with the rest of the system, [![\[Balbir Singh\]](https://static.lwn.net/images/conf/2017/lsfmm/BalbirSingh-sm.jpg)](<https://lwn.net/Articles/717624/>) though. There is [a patch series](<https://lwn.net/Articles/713035/>) under development to support these "coherent device memory nodes" (or CDMs) on Linux. 

There are still a number of questions about how such hardware should work with Linux. The desire is to provide selective memory allocation: user-space applications could choose whether memory should be allocated in normal or CDM memory. Reclaim needs to be handled carefully, though, since the kernel may not have a full view of how the memory on the CDM is being used. For obvious reasons, normal NUMA balancing needs to be disabled, or pages will be move into and out of the CDM incorrectly. When migrations are desired, they should be accelerated using DMA engines. 

The plan for CDM memory is to allocate it on the CPU, but then to run software using that memory on the CDM's processor. The device is able to access its own memory and normal system memory transparently via pointers. The hope is to migrate memory to the most appropriate node based on the observed usage patterns. Hansen noted that the NUMA balancing code in the kernel works fairly well, but most people still turn it off; will there really be a call for it in this setting? Singh responded that it can make a big difference; hints from the application can also help. 

Thus far, the patches include support for isolating the CDMs using the cpuset mechanism. But the system doesn't have enough information to do memory balancing properly yet. The zone lists have been split to separate out CDM memory; that serves to hide it from most of the system and avoid confusion with regular memory. At the end of the session, transparent hugepage migration was raised as another missing piece, but that topic was deferred for later discussion.
