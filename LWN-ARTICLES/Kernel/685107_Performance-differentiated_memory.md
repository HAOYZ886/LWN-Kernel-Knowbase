---
title: "Performance-differentiated memory"
url: https://lwn.net/Articles/685107/
date: "April 27, 2016"
category: Memory management
author: "By Jake Edge April 27, 2016 LSFMM 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

There are memory devices coming down the pike that have different performance characteristics than traditional DRAM. Linux will need to support these devices, but there is a question whether they should be treated as traditional memory that is managed by the kernel or if they should be presented as separate devices with their own drivers. Dan Williams led a plenary discussion on that topic on the second day of the Linux Storage, Filesystem, and Memory-Management Summit. 

[ ![\[Dan Williams\]](https://static.lwn.net/images/2016/lsf-williams2-sm.jpg) ](<https://lwn.net/Articles/685158/>)

Technologies like [3D XPoint](<https://en.wikipedia.org/wiki/3D_XPoint>) and [High Bandwidth Memory](<https://en.wikipedia.org/wiki/High_Bandwidth_Memory>) (HBM) perform differently from DRAM. These types of devices might also be mirrored or serve as caches for slower memory. So applications may need to know that there is a limited amount of high-performance memory available, with more, slower memory behind that cache. 

If the memory-management subsystem is to handle this type of memory, it could be classified as a memory zone or some kind of NUMA node. There will be a need to find this memory by its type and location, so there may be a need for a new type of memory, Williams said, rather than tracking it as a "crazy kind of NUMA node". 

High-performance computing (HPC) applications, databases, and other applications that know how to do their own buffer management just need the kernel to tell them what type it is and where it is. The kernel would then get out of the way and let the application manage that space. To Williams, that seems like a device. 

Other applications, though, simply want something sane done with that memory. They don't need strong guarantees about what type of memory is used. That seems more like a memory-management subsystem job to him. 

It comes down to whether the memory is tracked with [`struct page`](<https://lwn.net/Articles/565097/>) or not, Ted Ts'o said. Williams thinks that the memory would start off being managed as a device without page structures, but it could be handed off to the memory-management subsystem at some point, which would create the page structures at that point. 

For persistent memory, there is a use case for hypervisors or databases that don't need a filesystem. The persistent memory can be carved up, in a way similar to partitions, then handed out in huge regions to these applications. Keith Packard said that for his work (on [The Machine](<https://lwn.net/Articles/655437/>)), the plan is to put a filesystem on top of chunks of persistent memory. But some of that memory can also be hotplugged into the kernel and get page structures at that time. 

The device side of things is fairly well understood, Williams said. It presents a file or device that an application can `mmap()` into its address space. The problem comes when you want to get the memory-management subsystem involved. He asked, is there a need to have something besides NUMA nodes to direct applications to the different memory types? 

Rik van Riel said that there is some existing code to direct applications to certain types of memory. What is missing, though, is a way to evict pages from faster memory back to slower memory. Williams said that persistent memory can have swap memory associated with it, which might form the basis of a rudimentary migration strategy. But it seemed to him that what was really needed was to send some patches for discussion. 

The memory access patterns will need to be tracked for different regions, Van Riel said, so that memory management can make decisions on migration and placement. There is some information available from the CPU performance counters, but that will not cover memory accesses, so there will need to be a way to track processes that are using the wrong type of memory. 

There is also a need for a unified way for different architectures to describe this memory to user space. But Packard wondered if would make sense to wait and see what the applications actually need. For now, he plans to simply expose the hardware and let the application developers figure out what more they need.
