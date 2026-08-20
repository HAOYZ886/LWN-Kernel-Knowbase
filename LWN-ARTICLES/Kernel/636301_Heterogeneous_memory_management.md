---
title: Heterogeneous memory management
url: https://lwn.net/Articles/636301/
date: "March 13, 2015"
category: "Memory management-Heterogeneous memory management"
author: "By Jonathan Corbet March 13, 2015 LSFMM 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 13, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

Jérôme Glisse started an LSFMM 2015 memory-management track session on heterogeneous memory management (HMM) by noting that memory bandwidth for CPUs has increased slowly in recent years. There is little motivation for faster progress, since not many workloads sustain maximum memory bandwidth; instead, CPU access patterns are relatively random, and latency is usually the determining factor in the performance of any given workload. 

When one looks at graphical processing units (GPUs), the story is a bit different. Contemporary GPUs are designed for good performance with up to 10,000 running threads; to get there, they can have a maximum memory bandwidth that exceeds CPU-memory bandwidth by a factor of ten. Even so, a good GPU can saturate that bandwidth. GPUs, in other words, can do some things extremely quickly. 

Increasingly, Jérôme said, we are seeing systems where the CPU and the GPU are placed on the same die, both with access to the same memory. The GPU is useful for "light" gaming, user-interface rendering, and more. On such systems, most of the memory bandwidth is used by the GPU. 

The HMM code exists to allow the CPU and GPU to share the same memory and the same address space; it could eventually be useful for other devices with access to memory as well. The GPU gains software capabilities similar to those the CPU has; it runs its own page table, can incur page faults, and more. The key is to provide a way to manage the ownership of a given [![\[Jerome Glisse\]](https://static.lwn.net/images/conf/2015/lsfmm/JeromeGlisse-sm.jpg)](<https://lwn.net/Articles/636320/>) block of memory to avoid race conditions. And that is what HMM does; it provides a way to "migrate" memory between the CPU and the GPU, with only one side having access at any given time. If, say, the CPU attempts to access memory that currently belongs to the GPU, it will incur a page fault. The fault-handling code can then migrate the memory back and allow the CPU's work to proceed. 

Implementing this functionality requires the ability to keep page tables synchronized on both sides; that is done on the CPU side through the use of a memory-management unit (MMU) notifier callback. Whenever the status of a block of memory changes, the appropriate page-table invalidations can be done. There is one catch, though: to work properly, the notifier needs to be able to sleep, which is not something that MMU notifiers are currently allowed to do. That has been a sticking point for the acceptance of this patch so far. 

Andrew Morton jumped in to express some concerns about the generality of this system. GPUs are changing rapidly, he said; we could easily reach a point where, five years from now, nobody is using the HMM code anymore, but it still must be maintained. Jérôme responded that he believes the system is sufficiently general to be useful for GPUs, digital signal processors, and other devices for a long time. 

Jérôme finished up by saying that HMM support is needed in order to provide full, transparent GPU support to applications. The compiler projects are working on the ability to vectorize loops for execution on the GPU; when this works, applications will be able to use the GPU without even knowing about it. 

Rik van Riel asked if the group had any issues with the HMM code that needed discussion. Mel Gorman asked how many people had actually read the patch set; it turned out that not many had. Rik had reviewed an older version and didn't find any real issues with it. Andrew noted that there have not been a whole lot of reviews of the HMM code in general, and there do not appear to be many other users waiting in the wings. 

The session finished with some scattered discussion of various HMM details. How is the migration of anonymous pages to a device handled? The answer is that the device looks like a special type of swap file. The trick here is in handling of `fork()`; in this case, all of the relevant memory must be migrated back to the CPU first. Atomic access by the device is handled by mapping the relevant page(s) as read-only on the CPU; subsequent write faults look a lot like copy-on-write faults. It would be nice to be able to handle file-backed pages in the HMM system; that would require the creation of a special entry type in the page cache. That brings up a problem similar to the MMU-notifier issue: the filesystem code assumes that page-cache lookups are atomic, but, in this case, the code will need to sleep. It is not clear how to handle that one; adding HMM-specific code to each filesystem was mentioned, but that does not appear to be an appealing option.
