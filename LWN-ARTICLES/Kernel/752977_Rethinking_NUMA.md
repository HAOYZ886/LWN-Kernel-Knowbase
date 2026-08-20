---
title: Rethinking NUMA
url: https://lwn.net/Articles/752977/
date: "April 27, 2018"
category: "Memory management-NUMA systems; NUMA"
author: "By Jonathan Corbet April 27, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 27, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The non-uniform memory architecture (NUMA) was designed around the idea that there are two types of memory on complex systems: local (faster) and remote (slower). During the memory-management track of the 2018 Linux Storage, Filesystem, and Memory-Management Summit, Anshuman Khandual asserted that the situation has since become rather more complicated. Perhaps, he said, the time has come to rethink how we view NUMA systems. 

On upcoming hardware, Khandual said, there are memory interfaces that can deal with numerous types of memory, all of which ends up looking like DRAM. Memory can vary in parameters like bandwidth, persistence, latency, and power consumption. [![\[Anshuman
Khandual\]](https://static.lwn.net/images/conf/2018/lsfmm/AnshumanKhandual-sm.jpg)](<https://lwn.net/Articles/752983/>) Applications may want to take advantage of this diversity by, for example, putting an important but rarely accessed data structure in low-power memory, reserving the faster (and more power-hungry) memory for data that must be closer to hand. A lot of this kind of control can be achieved now with `mmap()`, but that approach leaves no room for integration with the memory-management subsystem. For example, there is no migration of pages between different memory types in response to the system workload. Things would work better if the kernel had a better understanding of memory attributes. 

There is, he said, an existing solution using the current NUMA abstraction: create new nodes to hold slower memory and set the distance value accordingly. Applications can then map memory into those zones if they want, but pages should not end up there by default. Special memory should not be used like normal memory. Matthew Wilcox replied that this might not always be the case; if the alternative is to put the system into an out-of-memory panic, it might be better to allocate pages from a slow zone. That is what happens now, Khandual said; with sufficient memory pressure, pages will be placed in the special zones — but, depending on the nature of those zones, that might not be desirable. 

Khandual suggested keeping more attribute information inside NUMA zones, perhaps tagging the memory with different zone or migration types. That would help to prevent implicit allocations in those zones. Beyond unwanted spillover, the simple fact is that node distance is not enough to capture the differences between different types of memory. For example, Jérôme Glisse said, a system with two GPUs may have a faster link between them; memory allocations on one GPU should fall back to the other if need be, but there is no way to express that in the kernel now. 

If some way is found to encapsulate memory attributes into NUMA nodes, there still needs to be a way to get that information out to user space so applications can make use of it. There was talk of a new sysfs interface, but there were also worries that it could grow too large on a system with a lot of nodes. Perhaps what is needed, Khandual suggested, is a new API to request memory with specific attributes. 

That suggestion concerned Dave Hansen, who said that this kind of API would require a lot of thought and is fraught with pitfalls. The original plans for NUMA support included a lot of options, but most of them turned out not to be needed in the real world. We are, he said, terrible at designing interfaces in general; there is no way that we would get it right when the hardware we are designing for is not even available yet. Instead, he said, the thing to do is to find the places where the current NUMA interface isn't working now, then build a case for small additions to the API when they make sense. But, to the extent that it is possible, it would be better to rely on the existing APIs for now. 

The session concluded with a warning to Khandual that, in typical memory-management fashion, he would be invited to the next five annual LSFMM events to give reports on how the work is progressing.
