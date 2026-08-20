---
title: The path to deprecating SPARSEMEM
url: https://lwn.net/Articles/974517/
date: "May 22, 2024"
category: "Memory management-Physical memory model"
author: "By Jonathan Corbet May 22, 2024 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 22, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The term "memory model" is used in a couple of ways within the kernel. Perhaps the more obscure meaning is the memory-management subsystem's view of how physical memory is organized on a given system. A proper representation of physical memory will be more efficient in terms of memory and CPU use. Since hardware comes in numerous variations, the kernel supports a number of memory models to match; see [this article](<https://lwn.net/Articles/789304/>) for details. At the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Oscar Salvador, presenting remotely, made the case for removing one of those models. 

The SPARSEMEM model, he said, is suited to systems with sparse memory — systems with large gaps in the physical address space. The newer SPARSEMEM_VMEMAP model also works on such systems; it makes life easier for the higher layers by virtually mapping physical memory in a way that makes it appear contiguous. Salvador said that the time for SPARSEMEM has passed, and that it was time to consider removing it in favor of using SPARSEMEM_VMEMAP instead. 

Michal Hocko immediately asked what the motivation was for the removal of SPARSEMEM. Salvador answered that it duplicates a lot of functionality with SPARSEMEM_VMEMAP. When the latter was initially introduced, developers did not convert all SPARSEMEM systems out of concern for the extra memory used for the virtual mapping. SPARSEMEM_VMEMAP also simply will not work on systems where the amount of physical memory exceeds the virtual address space. These concerns have abated in recent years, so a complete conversion to SPARSEMEM_VMEMAP can be considered; it would allow the removal of a fair amount of code. 

A participant agreed that the four architectures that only support SPARSEMEM — arm, mips, parisc, and sh — could be converted to SPARSEMEM_VMEMAP, though parisc might be better served with the simpler FLATMEM model. David Hildenbrand worried, though, that SPARSEMEM_VMEMAP could still be a problem for 32-bit architectures, which have limited virtual address spaces. Perhaps, he said, SPARSEMEM support could be dropped entirely for 32-bit systems; memory hotplugging, which had been one of the motivations for SPARSEMEM in the first place, is no longer supported there. Mike Rapoport, though, said that 32-bit Arm systems use SPARSEMEM to represent widely spread memory banks, a usage that is not related to hotplugging. Switching those systems to FLATMEM would require a lot of virtual address space that would have to come from the (already tight) vmalloc area. 

Hocko asked what problems are caused by SPARSEMEM; one of them, it seems is that SPARSEMEM complicates the addition of new hotplug features. He suggested just dropping hotplug support from SPARSEMEM, and not adding new features to it in general. Salvador, though, made it clear that he would rather remove the model entirely. 

Rapoport said that Arm systems can support a "sparse FLATMEM" model that would allow them to reduce the address-space usage; perhaps the other 32-bit architectures could do the same. That is a question that the various architecture maintainers would have to answer. 

Hocko concluded the session by saying that the removal could be a nice thing to try, since it would take out a lot of code. The first step would be to simply disable hotplug in the SPARSEMEM model. After that, it will be a matter of talking to architecture maintainers, trying to get each to move away from it.
