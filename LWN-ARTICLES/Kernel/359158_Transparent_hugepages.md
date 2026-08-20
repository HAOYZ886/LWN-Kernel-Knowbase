---
title: Transparent hugepages
url: https://lwn.net/Articles/359158/
date: "October 28, 2009"
category: "Huge pages; Memory management-Huge pages"
author: "By Jonathan Corbet October 28, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 28, 2009

Most Linux systems divide memory into 4096-byte pages; for the bulk of the memory management code, that is the smallest unit of memory which can be manipulated. 4KB is an increase over what early virtual memory systems used; 512 bytes was once common. But it is still small relative to the both the amount of physical memory available on contemporary systems and the working set size of applications running on those systems. That means that the operating system has more pages to manage than it did some years back. 

Most current processors can work with pages larger than 4KB. There are advantages to using larger pages: the size of page tables decreases, as does the number of page faults required to get an application into RAM. There is also a significant performance advantage that derives from the fact that large pages require fewer translation lookaside buffer (TLB) slots. These slots are a highly contended resource on most systems; reducing TLB misses can improve performance considerably for a number of large-memory workloads. 

There are also disadvantages to using larger pages. The amount of wasted memory will increase as a result of internal fragmentation; extra data dragged around with sparsely-accessed memory can also be costly. Larger pages take longer to transfer from secondary storage, increasing page fault latency (while decreasing page fault counts). The time required to simply clear very large pages can create significant kernel latencies. For all of these reasons, operating systems have generally stuck to smaller pages. Besides, having a single, small page size simply works and has the benefit of many years of experience. 

There are exceptions, though. The mapping of kernel virtual memory is done with huge pages. And, for user space, there is "hugetlbfs," which can be used to create and use large pages for anonymous data. Hugetlbfs was added to satisfy an immediate need felt by large database management systems, which use large memory arrays. It is narrowly aimed at a small number of use cases, and comes with significant limitations: huge pages must be reserved ahead of time, cannot transparently fall back to smaller pages, are locked into memory, and must be set up via a special API. That worked well as long as the only user was a certain proprietary database manager. But there is increasing interest in using large pages elsewhere; virtualization, in particular, seems to be creating a new set of demands for this feature. 

A host setting up memory ranges for virtualized guests would like to be able to use large pages for that purpose. But if large pages are not available, the system should simply fall back to using lots of smaller pages. It should be possible to swap large pages when needed. And the virtualized guest should not need to know anything about the use of large pages by the host. In other words, it would be nice if the Linux memory management code handled large pages just like normal pages. But that is not how things happen now; hugetlbfs is, for all practical purposes, a separate, parallel memory management subsystem. 

Andrea Arcangeli has posted [a transparent hugepage patch](<http://lwn.net/Articles/358904/>) which attempts to remedy this situation by removing the disconnect between large pages and the regular Linux virtual memory subsystem. His goals are fairly ambitious: he would like an application to be able to request large pages with a simple `madvise()` system call. If large pages are available, the system will provide them to the application in response to page faults; if not, smaller pages will be used. 

Beyond that, the patch makes large pages swappable. That is not as easy as it sounds; the swap subsystem is not currently able to deal with memory in anything other than `PAGE_SIZE` units. So swapping out a large page requires splitting it into its component parts first. This feature works, but not everybody agrees that it's worthwhile. Christoph Lameter [commented](<https://lwn.net/Articles/359183/>) that workloads which are performance-sensitive go out of their way to avoid swapping anyway, but that may become less true on a host filling up with virtualized guests. 

A future feature is transparent reassembly of large pages. If such a page has been split (or simply could not be allocated in the first place), the application will have a number of smaller pages scattered in memory. Should a large page become available, it would be nice if the memory management code would notice and migrate those small pages into one large page. This could, potentially, even happen for applications which have never requested large pages at all; the kernel would just provide them by default whenever it seemed to make sense. That would make large pages truly transparent and, perhaps, decrease system memory fragmentation at the same time. 

This is an ambitious patch to the core of the Linux kernel, so it is perhaps amusing that the chief complaint seems to be that it does not go far enough. Modern x86 processors can support a number of page sizes, up to a massive 1GB. Andrea's patch is currently aiming for the use of 2MB pages, though - quite a bit smaller. The reasoning is simple: 1GB pages are an unwieldy unit of memory to work with. No Linux system that has been running for any period of time will have that much contiguous memory lying around, and the latency involved with operations like clearing pages would be severe. But Andi Kleen [thinks this approach is short-sighted](<https://lwn.net/Articles/359184/>); today's massive chunk of memory is tomorrow's brief email. Andi would rather that the system not be designed around today's limitations; for the moment, no agreement has been reached on that point. 

In any case, this patch is an early RFC; it's not headed toward the mainline in the near future. It's clearly something that Linux needs, though; making full use of the processor's capabilities requires treating large pages as first-class memory-management objects. Eventually we should all be using large pages - though we may not know it.
