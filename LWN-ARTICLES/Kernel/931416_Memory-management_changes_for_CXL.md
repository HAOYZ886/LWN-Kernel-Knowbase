---
title: "Memory-management changes for CXL"
url: https://lwn.net/Articles/931416/
date: "May 12, 2023"
category: Compute Express Link CXL; Memory management
author: "By Jonathan Corbet May 12, 2023 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 12, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Kyungsan Kim began his talk at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) with a claim that the Compute Express Link (CXL) technology is leading to fundamental changes in computer architecture. The kernel will have to respond with changes of its own, including in its memory-management layer. Drawing on some experience gained at Samsung, Kim had a few suggestions on the form those changes should take — suggestions that ran into some disagreement from other memory-management developers. 

#### Requirements

CXL, he said, is creating real-world use cases for memory tiering, an architecture that separates memory with different performance characteristics and attempts to place each page in an appropriate tier. A classic example is placing "hot" (frequently accessed) pages in a fast, near tier, while cold pages could be placed more remotely — in slower memory attached via CXL, for example. Whether any given page is hot is, in the end, determined by its user, while the positioning is determined by the provider. His work has focused on the provider side, and has led to a set of requirements and proposals. 

[![\[Kyungsan Kim\]](https://static.lwn.net/images/conf/2023/lsfmm/KyungsanKim-sm.png)](<https://lwn.net/Articles/931432/>) The first of those requirements is that users should use NUMA node IDs to access CXL-hosted memory. That corresponds to how Linux has implemented access to such memory to date — by placing it into a separate CPU-less NUMA node. Those node IDs themselves do not reflect the distance of any given memory; a separate API is required for that. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Kim's second requirement is that the system should prevent unwanted migration of pages between nodes. As an example, he described a system running [zswap](<https://www.kernel.org/doc/html/latest/admin-guide/mm/zswap.html>), which "swaps" pages by compressing them and storing the result in RAM. A zswap configuration could reclaim pages from remote CXL memory and store the compressed result in fast, local memory, essentially promoting those pages. Michal Hocko responded that the solution to that specific problem was to fix zswap to preserve locality when compressing pages. David Hildenbrand added that, if pages are being put into zswap, they have already been determined to be cold, so it would be better to just put zswap-compressed pages in slow storage to begin with. 

Kim then presented his first proposal, which was to provide an explicit API to allocate different types of memory. In user space, this API would use the existing [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>), [`mbind()`](<https://man7.org/linux/man-pages/man2/mbind.2.html>), and [`set_mempolicy()`](<https://man7.org/linux/man-pages/man2/set_mempolicy.2.html>) system calls. So, for example, `mmap()` would gain a couple of new flags: `MMAP_NORMAL` to request a mapping in "normal", local memory, and `MMAP_EXMEM` to map into CXL-stored, remote memory. There would be similar new flags for the NUMA memory-policy system calls. Within the kernel, the new `GFP_NORMAL` and `GFP_EXMEM` flags would be used to explicitly request a given memory type. 

Notably, the defaults are different between the user-space and kernel interfaces. User space allocations could be placed in CXL memory unless the application explicitly specifies otherwise, but the kernel will only get that memory if it asks for it. Allowing unmovable kernel allocations to land in CXL memory could lead to problems if an attempt is made to unplug that memory in the future. 

Unplugging was the subject of Kim's next requirement: that the ability to unplug CXL memory must be maintained. Avoiding kernel-space allocations in CXL-attached memory is one step in that direction, but it does not avoid all problems. User-space memory, too, can end up being non-movable (and thus block unplugging) in some situations, he said, citing pages that are pinned for DMA as an example. Matthew Wilcox objected that, when user-space pages located in a movable zone are pinned, they are automatically migrated out of that zone first, so that problem should not actually exist. Hocko added that CXL memory should always be configured outside of `ZONE_NORMAL` to ensure that kernel allocations will not be placed there; it would not do to put kernel data structures in memory with potentially unbounded latency. 

Kim started into the fourth requirement, which was to reduce the number of CXL nodes that are visible to user space. Managing all of those nodes can become unwieldy. It would be better to, somehow, aggregate CXL resources into a single node. There was another requirement to be presented, but at this point the session had run over time, and the discussion came to an end. 

#### Zoning out

The subject was revisited in another session late in the conference, though Kim's final requirement was never presented. Meanwhile, it seems that some of what _was_ presented resulted in some hallway disagreements; in response, Dan Williams stood in front of the room and proclaimed his intent to mediate between Kim and his colleagues on one side, and the memory-management community on the other. The specific sticking point was Kim's proposal to add a new `mmap()` flag as described above, and especially to add a new memory zone (which wasn't covered in the earlier session, but would be needed to implement the flag). Neither of those things is done lightly, he said, but it can happen; similar changes were made for persistent memory. Williams said that he had "risked his career" to get [`MAP_SYNC`](<https://lwn.net/Articles/731706/>) into the kernel. 

[![\[Dan Williams\]](https://static.lwn.net/images/conf/2023/lsfmm/DanWilliams-sm.png)](<https://lwn.net/Articles/931556/>) First, though, Williams wanted to talk about the problem to be solved, and that involved a quick overview of how CXL-attached memory looks to a system. On these systems, the host's physical address space is divided into "windows"; this division is set up in the ACPI tables and does not change over the life of the system. Each window is a place where a CXL host bridge can be mapped; it is possible to map resources from more than one host bridge into a single window. 

The Linux kernel does "the simple thing" with host bridges for now. Attached memory is organized into performance classes, then a NUMA node is created for each class. Memory from multiple devices can be mapped into the same node if they all have similar performance. 

When it comes to using this memory, the primary need is to be able to bind processes to memory of one or more specific performance classes. At times, it may also be necessary to create bindings that _avoid_ a given performance class that might, for example, be too slow for most use. Processes would only be given memory from that class if they explicitly request it. The kernel also needs to avoid most CXL memory for its own data structures. There will also certainly be a need to migrate a process from one performance class to another at times. 

It is currently difficult to bind processes to a specific class of memory using just the NUMA API. The problem comes about when a new node comes online; running applications will not know that they are able to bind to that new node, and so will not make use of it. The solution, it was suggested in the room, is to bind to all _possible_ nodes, not just those that exist at the time. 

Kim returned to the front of this room at this point to make the case for a different approach. Internally, the kernel divides memory into zones with different characteristics. `ZONE_NORMAL` usually contains most memory, while `ZONE_MOVABLE` is meant to be restricted to allocations that can be moved at need, for example. Kim said that putting CXL-attached memory into its own zone might be a better way to reflect its performance characteristics than using NUMA nodes. Allocation policies are generally applied at the zone level, he said, so it is a more natural fit. 

Hocko disagreed, saying that zones are an internal memory-management concept and should not be exposed outside of the core code. The right place for CXL memory will be `ZONE_MOVABLE` most of the time. But Kim said that the existing zones are not well suited to CXL memory. `ZONE_NORMAL` is not pluggable, he said, while `ZONE_MOVABLE` does not allow pinning, which should be allowed for at least some CXL memory. `ZONE_DEVICE`, which is for memory hosted on attached devices, does not allow page allocation. 

Kim would, thus, like to add `ZONE_EXMEM` to handle the peculiarities of CXL memory, which go beyond variable performance characteristics. For example, there are other performance-related concerns; CXL memory is subject to link negotiation and quality-of-service throttling. There can be errors caused by connection problems; these tend not to arise with normal RAM. CXL memory has sharing and permission options that are not present with normal memory; it can also implement some types of asynchronous operations ("sanitize", for example) that the host need not wait for. 

Williams answered that asynchronous operations can be managed with normal memory as well, and that regular memory is certainly not immune to errors. Zones would just add complexity, he said, without enabling anything new. Kim, though, insisted that a zone-based solution would require fewer code changes to implement. 

Wilcox argued that exposing zones more widely is an idea that scares people; zones should be hidden within the memory-management subsystem. Nodes are sufficient, he added, to do what needs to be done. Hocko worried that a single zone would not suffice if that approach were taken; he wondered how many zones would be needed in the end. Williams added that the memory-management developers have been working on integrating high-bandwidth memory, which is also a whole new class of memory, but nobody has thought about adding a new zone for it. 

It was the end of the third day of the conference, and everybody was tired. As thoughts turned toward beer, Williams summarized the current state of the conversation. He has not heard, he said, that anybody (other than Kim) is convinced that nodes are not sufficient for CXL memory, but that the community could still be convinced otherwise. Doing so, though, would require a well-defined use case that cannot be handled with NUMA nodes. So the response to adding a new zone is "no" today, but could be changed in response to a compelling example of why that zone is needed. That, he said, is how `MAP_SYNC` came about — and it took a two-year discussion to get there. The plan to add a new zone for CXL-attached memory will have to clear a similar bar.
