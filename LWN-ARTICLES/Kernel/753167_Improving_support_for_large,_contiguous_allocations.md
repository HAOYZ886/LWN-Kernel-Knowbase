---
title: "Improving support for large, contiguous allocations"
url: https://lwn.net/Articles/753167/
date: "May 1, 2018"
category: "Memory management-Large allocations"
author: "By Jonathan Corbet May 1, 2018 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 1, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

Allocating chunks of memory that are both large and physically contiguous has long been a difficult thing to do in the kernel. But there are times where there is no alternative. Two sessions in the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit explored ways of making those allocations more reliable. It turns out that some use cases have a rather larger value of "large" than others. 

#### The contiguous memory allocator

Allocating large, physically contiguous blocks of memory becomes increasingly difficult over the life of the system as memory gets more fragmented. Most kernel code goes out of its way to avoid such allocations, but there are places where they are necessary; for such situations, the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>) can be used. 

In her session on CMA, Fedora kernel maintainer Laura Abbott said that it relies on the kernel's "page block" mechanism, and it needs both reclaim and compaction to work [![\[Laura Abbott\]](https://static.lwn.net/images/conf/2018/lsfmm/LauraAbbott-sm.jpg)](<https://lwn.net/Articles/753161/>) properly. That is because CMA counts on being able to move other allocations out of the way to create large blocks when the need arises. Page blocks are a relatively small management unit in most kernels, being the same size as the smallest huge page by default. Some trouble comes up on ARM systems, though, when 64KB pages are in use; at that page size, a page block holds 512MB. CMA requires regions to be page-block aligned, but creating a 512MB block for CMA on a memory-constrained system like a Raspberry Pi does not work well. 

The current workarounds are to either do without CMA, or to use 4KB pages, neither of which is entirely appealing. Using smaller pages, in particular, would require Fedora to ship two kernels using different page sizes, and distributors will go far out of their way to avoid supporting multiple kernels if at all possible. Matthew Wilcox said that it might be interesting to add the ability for the kernel to automatically choose the page size when it boots, but that would not be a straightforward thing to implement. 

One possible solution is to tie CMA to the `ZONE_MOVABLE` zone, which also would solve a number of accounting problems. CMA would still be bound by the page-block size, though. There was some talk of maybe reducing the page-block size, which would address this problem, but it might come at the cost of support for huge pages. Nonetheless, changing the size of page blocks is easily supported in the kernel now, so it might be the cleanest path to a short-term solution. At the end of the session, though, Michal Hocko noted that the page-block size is an arbitrary software construct; if page-block alignment requirements are causing trouble, "we've done it to ourselves", he said. 

#### Going larger

The following session, led by Mike Kravetz, made it clear that "large" is a relative term. While CMA tends to be used for allocations measured in megabytes, he is working with a Mellanox RDMA controller that performs best when given a 2GB physically contiguous buffer to work with. While CMA allocations are performed within the kernel, this buffer is allocated in user space and registered with the driver. Needless to say, there can be some challenges involved in making it possible for that allocation to succeed. 

The [hugetlbfs](<https://lwn.net/Articles/374424/>) mechanism supports "gigantic page" allocations; it can provide 1GB pages on the x86 architecture. Given the tricks involved in making that work, Kravetz said, the code that does [![\[Mike Kravetz\]](https://static.lwn.net/images/conf/2018/lsfmm/MikeKravetz-sm.jpg)](<https://lwn.net/Articles/753170/>) this in hugetlbfs probably shouldn't be there. But it is a useful technique, and other use cases, such as Intel's patch set that goes under the concise name of "[resource director technology cache pseudo locking](<https://lwn.net/Articles/752794/>)", are starting to pop up as well. 

Kravetz has been looking into alternatives. One of those was to add [a `MAP_CONTIG` option](<https://lwn.net/Articles/736170/>) to the `mmap()` system call that would cause the new mapping to be populated with physically contiguous pages. That turns out to give user space too much control over how allocations are made, though. 

So he is looking instead at [adding a new allocation function](<https://lwn.net/Articles/747170/>) called `find_alloc_contig_pages()`. This function, meant to be called from within the driver, replicates much of the behavior found in the hugetlbfs allocator. It requires that the system support page migration so that the huge pages can be cleared if needed. The memory it allocates is in the movable zone, which may well be a problem for buffers meant to be used with peripheral devices. 

The actual use case for this functionality lies in the Oracle database, which needs to register global areas for RDMA transfers. For hardware performance, a 2GB physically contiguous area is preferred. The CPU, though, needs to see this area as 2MB huge pages, which perform better than 1GB gigantic pages on current hardware. This memory could be allocated with hugetlbfs, Kravetz said, or the kernel could just allocate contiguous areas at boot. 

He was seemingly hoping for some guidance from the developers on which of these approaches might be the least ugly. The session ran out of time, though, without a definitive conclusion on the best way to obtain this kind of memory allocation.
