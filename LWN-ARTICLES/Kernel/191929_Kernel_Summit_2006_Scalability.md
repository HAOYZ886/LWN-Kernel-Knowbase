---
title: "Kernel Summit 2006: Scalability"
url: https://lwn.net/Articles/191929/
date: "July 19, 2006"
category: Scalability
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Scalability work has transformed the Linux kernel over the past years. Christoph Lameter took the stage to talk about the current state of Linux scalability. How far does Linux go now? 

His answer is that, if your hardware has a single memory bus, Linux will scale to somewhere between four and eight processors; after that, the memory bandwidth is simply not available. NUMA techniques are needed to create larger systems. On SGI's Altix systems, Linux runs well on systems with anywhere between 32 and 512 processors. 1024-processor systems work well, but there are signs of limits being reached. There is a 1024-node Altix system - with 4096 processors - which runs now, and which has been certified by SUSE. There is also a 1024-node blade system, with 8TB of memory, which works now - but it takes up to an hour to boot. 

The boot time is caused by the fact that much of the system initialization work is performed by a single processor. The boot processor does all of the device and memory probing - and this system has a _lot_ of both. If the initialization work could be parceled out, such that each processor (or, at least, one processor on each node) locates its own memory and devices, the boot process would be sped up considerably. 

There are occasional scalability problems encountered on systems with more than 64 processors. Certain data structures - radix trees, the dcache, inode locks - start to experience contention. One way to reduce that would be to start replicating some shared directories, such as the root and `/usr`, across NUMA nodes. Other problems include memory balancing across the system and hardware failures. On current systems, a hardware failure brings down the whole system; it is necessary to find a way to let individual nodes fail while keeping the rest of the system running. 

Future work - scaling to systems with 1024 to 4096 processors - includes a number of the usual suspects: trimming data structures, lockless algorithms, etc. Some structures grow as the product of the number of processors, nodes, and memory zones, so they get quite big on larger systems. Christoph's recent work, aimed at reducing the number of memory zones, is motivated by this problem. The need to replicate memory pages between nodes is growing. The current 4096-byte page size is increasingly causing performance problems; there may eventually need to be a way to increase it. It would also help if the scheduler could, when moving processes between nodes, take the location of the bulk of the process's pages into account. 

Scalability issues will always exist. But the challenge, at this point, lies above 1024 processors. That is a testament to the amount of work which has been done this far. 

  * [Next: DMA and IOMMU issues](<https://lwn.net/Articles/191931/>)
