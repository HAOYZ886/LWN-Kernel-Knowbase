---
title: More on fragmentation avoidance
url: https://lwn.net/Articles/159110/
date: "November 8, 2005"
category: "Hotplug-Memory; Memory management-Large allocations"
author: 
---

[Last week's article](<http://lwn.net/Articles/158211/>) on fragmentation avoidance concluded with these famous last words: 

But there are legitimate reasons for wanting this capability in the kernel, and the issue is unlikely to go away. Unless somebody comes up with a better solution, it could be hard to keep Mel's patch out forever. 

One thing which _can_ keep a patch out of the kernel, however, is opposition from Linus, and that is what has happened in this case. His [position](<https://lwn.net/Articles/159111/>) is that fragmentation avoidance is "totally useless," and he concludes: 

Don't do it. We've never done it, and we've been fine. 

The right solution, according to Linus, is to create a special memory zone on the (rare) systems which need to be able to free up large, contiguous blocks of memory. Kernel memory allocations would not be allowed in that zone, so it would only contain user-space pages. Those pages are relatively easy to move when the need arises, so most needs would be satisfied. A certain amount of kernel tuning would be required, but that is the price to be paid for running highly-specialized applications. 

This approach is not pleasing to everybody involved. Andi Kleen [noted](<https://lwn.net/Articles/159112/>): 

You have two choices if a workload runs out of the kernel allocatable pages. Either you spill into the reclaimable zone or you fail the allocation. The first means that the huge pages thing is unreliable, the second would mean that all the many problems of limited lowmem would be back. 

Others have noted that it can be hard to tune a machine for all workloads, especially on systems with a large number of users. Objections notwithstanding, it begins to look like active fragmentation avoidance is not likely to go into the 2.6 kernel anytime soon.
