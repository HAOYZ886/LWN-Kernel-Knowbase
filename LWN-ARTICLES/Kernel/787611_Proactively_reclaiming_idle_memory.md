---
title: Proactively reclaiming idle memory
url: https://lwn.net/Articles/787611/
date: "May 7, 2019"
category: "Memory management-Page replacement algorithms"
author: "By Jonathan Corbet May 7, 2019 LSFMM"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 7, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Shakeel Butt started his 2019 Linux Storage, Filesystem, and Memory-Management Summit session by noting that memory makes up a big part of the total cost of equipping a data center. As a result, data-center operators try to make the best use of memory they can, generally overcommitting it significantly. In this session, Butt described a scheme in use at Google to try to improve memory utilization; while the need for the described functionality was generally agreed upon, the developers in the room were not entirely happy with the solution presented. 

Overcommitting memory increases total utilization, but it comes at a cost: systems experience memory pressure and end up having to reclaim pages. Direct reclaim (where a process that is allocating memory has to do some of the work of freeing up memory used by others) is particularly problematic since it can introduce surprising delays; it reduces the isolation between users. The solution to this problem, he said, is to seek out and reclaim idle pages before memory gets tight. 

To this end, systems in the data center have been supplemented with slower (cheaper) memory, which can take any of a number of forms, including persistent memory, in-memory compression, or a real swap device. The system manages this memory in a way that is transparent to users. Then, idle [![\[Shakeel Butt\]](https://static.lwn.net/images/conf/2019/lsfmm/ShakeelButt-sm.jpg)](<https://lwn.net/Articles/787612/>) memory (pages that have not been accessed for some time) are located and pushed out to this slower memory. Butt said that, across the Google data center, about 32% of memory can be deemed to be idle at any given time. If that memory is reclaimed after two minutes of idle time, he said, about 14% of it will be refaulted back in; the rest is better used by somebody else. 

At this point, one might wonder why Google doesn't just use the kernel's existing reclaim mechanism, as implemented by the `kswapd` kernel thread. It "kind of works", he said, but is based on watermarks (keeping a certain percentage of memory free) rather than on idleness. `kswapd` also tries to balance memory usage across NUMA nodes, which is not useful in this setting. Finally, Butt said, `kswapd` is built on a large set of complicated heuristics, and he doesn't want to try to change them. 

So, instead, Google has put resources into a mechanism for tracking idle pages. There is a system in place now that is based on a user-space process that reads a bitmap stored in sysfs, but it has a high CPU and memory overhead, so a new approach is being tried. It is built on a new kernel thread called `kstaled`; it tracks idle pages using page flags, so it no longer adds memory overhead to the system, but it still requires a relatively large amount of CPU time. The new `kreclaimd` thread then scans through memory, reclaiming pages that have been idle for too long. 

The CPU cost is not trivial; it increases linearly with the amount of memory that must be tracked and with the scan frequency. On a system with 512GB of installed memory, one full CPU must be dedicated to this task. Most of this time is spent walking through the reverse-map entries to find page mappings. This has been improved by getting rid of the reverse-map walk and creating a linked list of mid-level (PMD) page tables; that reduced CPU usage by a factor of 3.5. Removing the scanning from `kreclaimd` in favor of a queue of pages passed in from `kstaled` gave another significant reduction. 

Butt said that he would like to upstream this work; it is not something that can be handled in user space. Rik van Riel noted that, even with the performance improvements that have been made, this system has scalability problems. Johannes Weiner asked why Google was reimplementing the tracking that is already done by the memory-management subsystem's least-recently-used (LRU) lists. Like the LRUs, this new mechanism is trying to predict the future use of pages; it might be nice to have, he said, but it is "crazy expensive". Butt replied that Google was willing to pay that cost, which was less than having the system go into direct reclaim. 

Weiner continued, saying that Facebook has faced the same issue. There, every workload must be containerized, and users are required to declare how much memory they will need. But nobody actually knows how much memory their task will require, so they all ask for too much, leading to the need to overcommit and reclaim issues. The solution being tried there is to use [pressure-stall information](<https://lwn.net/Articles/759781/>) to learn when memory is starting to get tight, then chopping the oldest pages off the LRU list. If the refault rate goes up, pages are reclaimed less aggressively. This solution, he said, yields reasonable results at a much smaller CPU cost. 

Discussion continued for a bit, but the general consensus was that, while this sort of proactive reclaim would be useful for a number of users, the cost of this particular solution was too high to consider merging it upstream.
