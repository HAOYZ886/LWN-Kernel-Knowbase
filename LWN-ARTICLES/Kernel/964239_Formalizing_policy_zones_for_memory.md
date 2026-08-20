---
title: Formalizing policy zones for memory
url: https://lwn.net/Articles/964239/
date: "March 5, 2024"
category: "Memory management-Huge pages"
author: "By Jonathan Corbet March 5, 2024"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 5, 2024

The kernel's memory-management subsystem is built on the concept of "zones", which were initially added to describe the physical characteristics of the memory pages contained within them. Over time, zones have taken on more of a policy-related role as well. With a patch set called [THP allocator optimizations](<https://lwn.net/ml/linux-mm/20240229183436.4110845-1-yuzhao@google.com/>), Yu Zhao has set out to better define the role of policy-related zones on the path toward adding two more of them, with the ultimate purpose of improving the kernel's support for transparent huge pages (THPs). 

#### The history zone

A bit of background might help to set the context for this patch series. 

The earliest x86 systems that ran Linux talked to peripheral devices via the ISA bus, which was only able to address 16MB of memory, using 24-bit addresses. Even in those days, computers often had more memory than that, meaning that some memory could not be used by ISA devices for DMA I/O operations. That made life inconvenient for any device drivers that might happen to allocate a DMA buffer in memory inaccessible by the devices they were trying to control. 

To avoid this problem, the kernel added the `GFP_DMA` allocation flag in 1.1.68, in November 1994. That flag didn't actually _do_ anything until 1.1.69, and was not used until the Buslogic SCSI driver added support in 1.1.73. This flag asked the memory-management subsystem to allocate memory within the physical address range that was reachable by ISA devices, so that driver authors no longer needed to worry when allocating buffers. 

At that time, `GFP_DMA` simply caused the allocator to skip over memory that was not suitably located. That worked, but life, as is its wont, became more complicated. The kernel, in the early days, could only manage a bit less than 1GB of physical memory; the 32-bit address space, as partitioned by the kernel, simply could not contain any more. That nearly 1GB, of course, ought to have been enough for anybody. But users can be rather strident when it comes to being able to use all of the memory they paid for, so kernel developers came under pressure to find a solution. 

That solution was "high memory", which could be allocated to user-processes but could not be directly addressed by the kernel without an explicit (and temporary) mapping step. High memory was added to 2.3.23pre3 in late 1999, adding another type of memory that had to be taken into account; memory used by the kernel usually could not be placed there. To deal with these constraints, the concept of "memory types" was added to 2.3.23pre5. That concept was then formalized into "zones" in the 2.3.27 release in November 1999. At that time, there were three zones: `ZONE_DMA`, `ZONE_NORMAL`, and `ZONE_HIGHMEM`. 

An important aspect of the zone system is that the zones were organized in an order that allowed for a straightforward fallback mechanism. Any allocation that could be served out of `ZONE_HIGHMEM` could also be satisfied from either of the other two zones, and `ZONE_NORMAL` allocations could fall back to `ZONE_DMA`. That property still holds in current kernels. 

The 2.6.15 kernel, released in January 2006, saw the addition of `ZONE_DMA32`, wedged between `ZONE_DMA` and `ZONE_NORMAL`, to meet the needs of devices that could only handle 32-bit DMA addresses. In 2.6.23 (October 2007), `ZONE_MOVABLE` was added, and `ZONE_DEVICE` showed up in 4.3 (November 2015) to describe CPU-addressable memory installed on peripheral devices. 

To an extent, all of these zones describe physical characteristics of the memory involved, mostly relating to addressability. `ZONE_MOVABLE` is a bit of an exception, though. It describes memory where, with luck, all of the contents can be moved elsewhere if need be. User-space pages, for example, can be migrated elsewhere in memory; the page-table entries will be updated accordingly, and user space will never know that anything changed. This zone was added to support hotplug memory that could be added to (or removed from) a system at any time. If memory is to be removed from a system, it is somewhat important to shift all of the contents of the memory elsewhere first. 

The hotplug use case still exists, but `ZONE_MOVABLE` has become more of an expression of memory-management policy. If all of the movable pages are put in the same region of memory, it is relatively easy to reclaim some of that memory to, for example, create large, physically contiguous ranges. So now `ZONE_MOVABLE` is used as a way of separating movable and non-movable allocations, and provides important support for, among other things, the [contiguous memory allocator](<https://lwn.net/Articles/486301/>). The important point here is that `ZONE_MOVABLE` can be placed anywhere in physical memory; it is not driven by the physical characteristics of that memory, especially on the large majority of systems where hotplugging is not in use. 

In current kernels, the zone hierarchy is: 

  * `ZONE_DMA`
  * `ZONE_DMA32`
  * `ZONE_NORMAL`
  * `ZONE_HIGHMEM`
  * `ZONE_MOVABLE`
  * `ZONE_DEVICE` 

Note, though, that not all zones are present on all systems; current 64-bit systems, for example, have no need for `ZONE_HIGHMEM`. 

#### Explicit policy zones

Zhao has come to the conclusion that more can be done with the idea of a memory zone as an aid to allocation policy. Specifically, he argues that there may be a use for a couple of other policy zones (which are often called "virtual zones" in the code itself). To that end, his patch series adds: 

  * `ZONE_NOSPLIT`, which is a zone where contiguous blocks of pages cannot be split below a given size. It exists to help the system maintain large blocks of memory (to use for transparent huge pages, among other things) without having to go through a continual process of compaction. 
  * `ZONE_NOMERGE` also has the minimum block-size property, but also disallows the merging of blocks of pages into bigger groups; as a result, it can only hold chunks of a single size. 

These zones, which are placed after `ZONE_MOVABLE`, maintain the fallback hierarchy: a `ZONE_NOMERGE` allocation can also be satisfied from `ZONE_NOSPLIT`, or from one of the lower zones if that fails. Allocations from these zones must be movable, so a fallback to `ZONE_MOVABLE` is possible as well. 

The idea behind these zones is to make the allocation of transparent huge pages as efficient as possible. They will prevent the splitting of huge pages, which will keep the kernel from having to go through the effort of reassembling them later. Much of the work that currently goes into compaction could become unnecessary. 

In a sense, this work can be seen as a sort of compromise between those who would like to see Linux use a larger page size overall and those who worry about the associated internal-fragmentation cost. `ZONE_NOMERGE` creates something close to a second native page size for the kernel, making larger pages available in situations where they make sense while still keeping smaller pages available. 

Internal fragmentation can still be a problem with transparent huge pages, though; a process may have such a page allocated, but only be using a small portion of the memory within it. Current kernels will try to respond to this situation by splitting huge pages back into base pages, allowing the unused parts to be reallocated elsewhere. 

Splitting clearly cannot happen with pages located in (or above) `ZONE_NOSPLIT`; that is exactly the policy that zone exists to enforce. Instead, Zhao's patch set introduces the concept of "shattering" huge pages. If a page is shattered, its contents are migrated (copied) to smaller pages located in a suitable zone; once that process completes, the original huge page, which remains intact, can be allocated to another use. Shattering is more expensive than splitting; Zhao sees it as an appropriate cost for a process not properly using its memory; from [the changelog](<https://lwn.net/ml/linux-mm/20240229183436.4110845-3-yuzhao@google.com/>): ""In retail terms, the return of a purchase is charged with a restocking fee and the original goods can be resold"". 

Another claimed advantage of `ZONE_NOMERGE` is that it facilitates [huge vmemmap optimization (HVO)](<https://www.kernel.org/doc/html/next/mm/vmemmap_dedup.html>), which was [covered here](<https://lwn.net/Articles/839737/>) in 2020. In short, this trick allows the kernel to recover the memory used to hold `page` structures for many of the pages in a huge page. In a system where a lot of huge pages are in use, this optimization can save a significant amount of memory. In current kernels, HVO can only be used with the hugetlbfs mechanism, which is not transparent and is normally only used in specialized situations. `ZONE_NOMERGE` pages are organized in fixed blocks, though, like hugetlbfs pages, so it becomes easy to use HVO with them. 

The patch set is in an early stage; among other things, it does not have any sort of benchmark results showing the advantages of this new machinery. Review comments from other developers are just beginning to come in; it is a significant chunk of work and will take some time to digest. It is likely to be a discussion topic at the [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) in May. The new zones will thus probably not land in the kernel in the near future, but their advantages might prove compelling in the longer term.
