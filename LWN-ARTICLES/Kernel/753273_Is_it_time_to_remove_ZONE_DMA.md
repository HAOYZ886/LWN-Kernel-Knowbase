---
title: "Is it time to remove ZONE_DMA?"
url: https://lwn.net/Articles/753273/
date: "May 4, 2018"
category: Memory management
author: "By Jonathan Corbet May 4, 2018 LSFMM"
---

By **Jonathan Corbet**  
May 4, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The DMA zone (`ZONE_DMA`) is a memory-management holdover from the distant past. Once upon a time, many devices (those on the ISA bus in particular) could only use 24 bits for DMA addresses, and were thus limited to the bottom 16MB of memory. Such devices are hard to find on contemporary computers. Luis Rodriguez scheduled the last memory-management-track session of the 2018 Linux Storage, Filesystem, and Memory-Management Summit to discuss whether the time has come to remove `ZONE_DMA` altogether. 

Rodriguez, however, was late to his own session, so the developers started discussing the topic without him. It's not clear that any modern devices still need the DMA zone, and removing it would free one precious page [![\[Luis Rodriguez\]](https://static.lwn.net/images/conf/2018/lsfmm/LuisRodriguez-sm.jpg)](<https://lwn.net/Articles/753275/>) flag. Any requests with the `GFP_DMA` flag could be redirected to the zone for the contiguous memory allocator (CMA) which, in turn, could be given the bottom 16MB of memory to manage. Matthew Wilcox asked whether the same thing could be done with `ZONE_DMA32`, used for devices that can only DMA to 32-bit addresses, but it is not possible to allocate all of the lowest 4GB of memory to that zone, since it would exclude kernel allocations. 

It was noted in passing that the POWER architecture uses `GFP_DMA` extensively. It doesn't actually need it, though; the early POWER developers had misunderstood the flag and thought that it was needed for any memory that would be used for DMA. 

At this point, Rodriguez arrived and presented his case. He noted that the existence of `ZONE_DMA` causes an extra branch to be taken in every memory allocation call. Perhaps removing the zone could improve performance by taking out the need for those branches. It's not clear that performance would improve all that much, but the developers would be happy to be rid of this ancient zone regardless. 

The problem is that quite a few drivers are still using `ZONE_DMA`, even if a number of them don't really need it. The SCSI subsystem was mentioned as having a number of allocations using it. Wilcox suggested that perhaps the drivers still using `ZONE_DMA` could be moved to the staging tree; they could then either be fixed and moved back or just removed entirely. A look at the list of affected drivers (which can be found in [this summary of the session](<https://lwn.net/Articles/753274/>) posted by Rodriguez) suggests that just deleting them is probably not an option, though. 

More work will be needed to determine the real effects of changing this zone, and of possibly redirecting it into the CMA zone instead. But its removal would simplify the memory-management subsystem, so there is motivation for the developers to do the necessary research.
