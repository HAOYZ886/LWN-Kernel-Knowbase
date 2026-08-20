---
title: Improving access to physically contiguous memory
url: https://lwn.net/Articles/787748/
date: "May 8, 2019"
category: "Memory management-Large allocations"
author: "By Jonathan Corbet May 8, 2019 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 8, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

For years, kernel developers have been told to avoid allocating large chunks of physically contiguous memory; as the system runs and memory becomes fragmented, satisfying such allocations becomes increasingly difficult. But, as Zi Yan pointed out in a memory-management track session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, there are times when contiguous memory is useful. In this session, the memory-management developers discussed ways to make such allocations more likely to succeed. 

There are a lot of uses for physically contiguous memory, Yan said. The use of huge (2MB) or gigantic (1GB) pages improves performance by reducing the demands on the CPU's translation lookaside buffer (TLB). But it also turns out that many high-bandwidth peripheral devices also prefer physically contiguous memory; those devices, too, have TLBs that can run out of space. A device must respond to a TLB miss by walking through its internal page tables, which tends to be rather slower than when the CPU does it. 

There are three ways of allocating large physically contiguous chunks now. One is the [libhugetlbfs](<https://lwn.net/Articles/374424/>) virtual filesystem, but it requires that memory be set aside for that purpose at boot time. Users may well get the size wrong, and there is no interface to allocate memory from libhugetlbfs inside the kernel. The second method, transparent [![\[Zi Yan\]](https://static.lwn.net/images/conf/2019/lsfmm/ZiYan-sm.jpg)](<https://lwn.net/Articles/787751/>) huge pages, suffers from fragmentation and must occasionally do compaction of system memory. It also depends on the kernel's buddy allocator, meaning that it is limited to buffers of size `MAX_ORDER`, which is normally set to eleven, meaning a maximum size of 2048 4096-byte pages. It thus cannot even come close to providing gigantic pages. Finally, there is [`alloc_contig_range()`](<https://elixir.bootlin.com/linux/v5.1/source/mm/page_alloc.c#L8187>), which is used with the [CMA memory allocator](<https://lwn.net/Articles/486301/>). This functionality is not available to user space, though. 

Dave Hansen pointed out that libhugetlbfs can now be resized at run time, eliminating one of the concerns there. Andrea Arcangeli thought it might be useful to be able to allocate transparent huge pages from libhugetlbfs, and that it might not be that hard to implement. It seems that the real problem isn't necessarily the ability to allocate large chunks, but concerns about the time required to do so, since compaction may be required to make such a chunk available. There was some general discussion on quick allocation of higher-order chunks of memory without any conclusions. 

Mel Gorman said that allocation latency for large chunks was an old problem. The kernel used to run into multi-second stalls when faced with such requests, but that was "a long, long time ago". Until somebody has observed and nailed down problems with current kernels, he sees those problems as being mostly hypothetical. Transparent huge pages have been through a few implementations since the 3.0 kernel; there is a lot of old information out there on the net that has not caught up with the current state of affairs. So anybody who is having trouble with large allocations should observe their systems, enable tracepoints, and report the results — all with a current, upstream kernel. "Don't handwave this one", he said, and he cautioned that he would not lose sleep over reports about problems on enterprise kernels. 

Yan was not finished, though: he had a proposal for a new mechanism to defragment virtual memory areas (VMAs) after memory has been allocated. This would be done by finding pairs of pages that could be exchanged in a way that would improve the situation. Unlike the kernel's `khugepaged` thread, it would defragment in place rather than allocating huge pages up front and moving data into them. The page-exchange idea caused a few raised eyebrows in the room; it seemed overly complex for the problem at hand. The advantage of this approach, Yan said, is that it doesn't require any pages to be allocated; data is exchanged between pages by copying word-by-word through the CPU's registers. 

Gorman asked why anybody should bother avoiding allocation of temporary pages; Yan said it might be worthwhile for larger pages, but the group was unconvinced. When Gorman asked about performance measurements, Yan replied that this exchange was faster than simply migrating two pages, but it was not clear why. Hansen asked what the overall problem was that was being solved, Yan said that it is a way to obtain 1GB gigantic pages without needing to change the kernel's `MAX_ORDER` parameter. The group was not convinced that this would be beneficial, though; current CPUs have tiny TLBs for gigantic pages, and nobody has been able to measure any real performance gain from using them. 

There was a bit of back-and-forth between Yan (who works for NVIDIA) and another NVIDIA developer in the room. It seems that there might, maybe, somehow, be interest within the company in using 1GB pages for better performance in some not-yet-developed product far in the future. Maybe. But naturally nobody could actually talk about any such products, and kernel developers have little interest in trying to support them. 

The session came to an end with Gorman saying that there was no reason to add new infrastructure for this purpose; `khugepaged` is there already. If the kernel's page-migration logic is too slow, the right thing to do is to make it faster rather than circumventing it. For example, he said, there is no effort made to batch migration work currently; it is a "stupid" implementation. There is a lot of low-hanging fruit that should be fixed before thinking about adding a whole new set of machinery.
