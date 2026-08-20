---
title: Memory controller performance improvements
url: https://lwn.net/Articles/1016856/
date: "April 17, 2025"
category: "Memory management-Control groups"
author: "By Jonathan Corbet April 17, 2025 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 17, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The kernel's memory controller works within the control-group mechanism to enforce memory-usage limits on groups of processes. This component has often had performance problems, so there is continual interest in optimizing it. Shakeel Butt led a session during the memory-management track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit to look at the current state of the memory controller and what can be done to reduce its overhead. 

Last year, he began, he [ran a session](<https://lwn.net/Articles/974575/>) dedicated to the deprecation of the version-1 memory controller, which is key to the never-ending goal of removing support for version-1 control groups in general. Since then, the code for this support has been moved to its own file and is disabled by default. Should somebody enable and use it, they will see deprecation warnings. 

[![\[Shakeel Butt\]](https://static.lwn.net/images/conf/2025/lsfmm/ShakeelButt-sm.png)](<https://lwn.net/Articles/1016858/>) There is still a lot to do with the version-2 controller, of course. First on his list was the reparenting of memory on the least-recently-used (LRU) lists. This is needed to handle the case when the last process within a control group exits; normally the control group itself would be deleted at that point, but that cannot happen if there are still pages in memory that are charged to the group. That has led to a longstanding "zombie control groups" problem, often wasting significant amounts of memory. Reparenting will find those pages and move their charging up the control-group hierarchy, allowing the empty groups to be deleted. 

A longstanding request is for swap accounting, which is supported in version 1 but not version 2\. Somebody in the audience pointed out that Google wants this feature. There are requests for better information on memory headroom — the amount of memory that is available for a control group to use or, in other words, how much more memory can be used before the group runs out. User-space management systems can use that information to decide whether to schedule a new job. 

Butt wondered whether the `memory.high` limit, which sets the point at which allocation requests will be forced into direct reclaim, is useful. He said he has tried using it for out-of-memory avoidance, but it is ineffective for multi-threaded applications and too nondeterministic in general. The group had no comments to offer on that point. 

There are, he said, two performance-critical operations within the memory controller: statistics maintenance and usage accounting. The controller currently maintains over 100 different counters. One of those, the actual memory usage of the group, must always be exact so that the limits can be enforced right away. The rest can be updated a bit more lazily. 

On the charging side, every allocation leads to a new charge for the current control group; that involves checking current usage against the limits and might trigger responses like direct reclaim, out-of-memory handling, or allocation failures. All of this work happens at every level in the control-group hierarchy, which can make allocations slow. There is a per-CPU charge cache, which can help to avoid some of this work; it is reasonably effective, Butt said, when a process remains on the same CPU. It is less so for networking applications where each incoming packet can cause processes to move between CPUs. 

He has [sent out a proposal](<https://lwn.net/ml/linux-mm/20250307055936.3988572-1-shakeel.butt@linux.dev/>) for a per-CPU cache that handles multiple control groups. The patch is oriented toward networking workloads now, but he wants to generalize it over time. He is wondering how many control groups should be cached, though, whether that number should be static, and what the eviction policy should be. Michal Hocko asked whether it might be better to do without the cache entirely, relying on atomic operations to do the accounting; if nothing else, that would provide a performance baseline that could be tested against. Butt agreed to try that approach. 

The charging of kernel allocations, he continued, is done per-byte as individual objects are allocated from (or returned to) the slab allocator. There is a per-CPU byte-charge cache that speeds this accounting. He thinks it could be further improved by ending the practice of disabling interrupts while this charging happens and using [local locks](<https://lwn.net/Articles/828477/>) instead. Perhaps [sheaves](<https://lwn.net/Articles/1010667/>) could be used to batch the accounting for freed objects; that would only help in cases where most of the objects come from the same control group, though. He is also considering keeping charged objects in per-CPU slabs that could be reused if possible. 

Returning to the statistics, he repeated that there are over 100 different metrics maintained for each control group. There are two sides to the infrastructure that maintains these statistics; the updating side is fast, while the reading side is slower. It uses a per-CPU data structure for updates; to read the statics, it is necessary to pass over all the CPUs to flush and aggregate the data. 

This all could perhaps be optimized, he said, by tracking updates and skipping the all-CPU flushing if the numbers are within a threshold. The kernel could also perform occasional flushing asynchronously to avoid an expensive, synchronous operation at read time. Mathieu Desnoyers asked why these statistics aren't just managed with simple, per-CPU counters rather than with this complex infrastructure; Butt answered that the complexity comes from the need to manage the counters at all levels of the control-group hierarchy. 

He repeated that the read operation is expensive due to the flushing requirement; high-frequency readers are especially affected by this. One of the problems is that reading a single counter causes all 100+ of them to be flushed. He is currently working on isolating each statistic from the others to reduce that cost. 

As time ran out, a participant asked whether the controller could flush the statistics more lazily and provide a number to user space indicating how stale they are at any given time. A separate system call could then be provided to force a flush when user space needs current numbers. Butt said that the idea has been discussed. Hocko said that much of the current infrastructure has been well optimized, so adding an interface that would be allowed to return older data might indeed be the right next step.
