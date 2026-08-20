---
title: "Solutions for direct-map fragmentation"
url: https://lwn.net/Articles/894557/
date: "May 12, 2022"
category: "Memory management-Direct map"
author: "By Jonathan Corbet May 12, 2022 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 12, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

The kernel's "direct map" makes the entirety of a system's physical memory available in the kernel's virtual address space. Normally, huge pages are used for this mapping, making it relatively efficient to access. Increasingly, though, there is a need to carve some pages out of the direct map; this splits up those huge pages and makes the system as a whole less efficient. During a memory-management session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Mike Rapoport led a session on direct-map fragmentation and how it might be avoided. 

Rapoport started by saying that the direct-map fragmentation problem is specific to the x86 architecture at this point; some other architectures cannot fragment their direct map at all. There are a number of activities that can lead to direct-map fragmentation, including [allocations for BPF programs](<https://lwn.net/Articles/892743/>), various [secret-memory mechanisms](<https://lwn.net/Articles/865256/>), and virtualization technologies like [SNP](<https://www.amd.com/system/files/TechDocs/SEV-SNP-strengthening-vm-isolation-with-integrity-protection-and-more.pdf>) and [TDX](<https://www.intel.com/content/www/us/en/developer/articles/technical/intel-trust-domain-extensions.html>). Other changes envisioned for the future, including the [permission `vmalloc()` API](<https://lwn.net/ml/linux-mm/20201120202426.18009-1-rick.p.edgecombe@intel.com/>) [![\[Mike Rapoport\]](https://static.lwn.net/images/conf/2022/lsfmm/MikeRapoport2-sm.png)](<https://lwn.net/Articles/894565/>) and using [protection keys supervisor (PKS) to protect page tables](<https://lwn.net/Articles/894531/>), will make things worse. As more subsystems carve pieces out of the direct map, the performance of the system will decline; this is an outcome worth avoiding. 

Rapoport's proposal is to coalesce these various uses into a single region of memory as a way of minimizing the fragmentation they create. Once a huge page has been split for carved-out memory, further requests for such memory should be satisfied from the same huge page, if possible. To that end, he suggests adding a new GFP flag (`__GFP_UNMAPPED`) so that normal page-allocator calls can be used to obtain memory that has been removed from the direct map. Callers using this flag would have to map the allocated memory in whatever way makes sense for their use case. A new migration type (`MIGRATE_UNMAPPED`) would be added to prevent this memory from being accidentally migrated back into direct-mapped memory. He has posted [a patch set](<https://lwn.net/ml/linux-mm/20220127085608.306306-1-rppt@kernel.org/>) implementing this idea in a prototype form; it "kind of works", he said. 

Michal Hocko said that using the page allocator might not be the best approach; it will be adding overhead to highly optimized fast paths for a rare case. Mel Gorman agreed that using the page allocator was overkill, creating a special case for a single user. Rapoport's addition of a separate migration type, he added, would end up fragmenting memory anyway because those pages cannot be moved. Rapoport answered that, in a long-running machine, direct-map fragmentation is inevitable, leading Gorman to answer that he does not want to see the extra complexity added to the page allocator to address a problem that will still happen. 

An alternative, Rapoport said, would be to have a separate allocation mechanism that sits next to the page allocator. In this case, each user would have their own cache, which is a less attractive option. But Gorman replied that migration types are not free either; each new one adds a set of linked lists and increases the size of the page-block bitmap. A better solution, he said, might be a special slab cache. 

David Hildenbrand said that, in his role working on memory hotplug, he hates memory that is not movable; Rapoport's proposal would create more unmovable memory and make the problem worse. Rapoport said that his patch tries to avoid movable zones when performing unmapped allocations, which should minimize the problem. Hocko repeated, though, that the page allocator is not the best place to make this type of allocation; users "count every CPU cycle" for memory allocations, and any extra overhead there is unwelcome. It would be better to build something like a slab allocator on top of the page allocator, he said. 

At the end of the session, Rapoport said that he would try to create some sort of slab-like solution. Vlastimil Babka cautioned that the existing slab allocator cannot be used for BPF programs; the slab allocator hands out objects of the same size, but every BPF program is different. Rapoport concluded by saying he wasn't sure how to solve all of the problems, but would be making the attempt soon.
