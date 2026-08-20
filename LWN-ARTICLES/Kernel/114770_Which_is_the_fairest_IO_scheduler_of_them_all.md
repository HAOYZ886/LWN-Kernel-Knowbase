---
title: "Which is the fairest I/O scheduler of them all?"
url: https://lwn.net/Articles/114770/
date: "December 8, 2004"
category: "Block layer-IO scheduling; CFQ IO scheduler; Elevator; IO scheduler"
author: 
---

Certain parts of the kernel, it seems, can be tweaked forever; I/O schedulers would count as one of those parts. Linux has three of them currently (plus a no-op scheduler), and its block I/O performance is generally quite good. But that doesn't mean it can't be improved. 

Jens Axboe recently decided to do some more hacking on his "completely fair queueing" (CFQ) scheduler; the result is the new [time-sliced CFQ scheduler](<https://lwn.net/Articles/113869/>), which has since seen a ~~[second](<https://lwn.net/Articles/114273/>)~~ ~~[third](<https://lwn.net/Articles/114379/>)~~ [fourth](<https://lwn.net/Articles/114734/>) revision. The CFQ scheduler has always tried to divide the bandwidth of each block device fairly among the processes performing I/O to that device; the time-sliced version goes further by giving each process exclusive access to the device for a period of time. 

In particular, the time-sliced scheduler picks a process, and dispatches only that process's requests to the device for some tens of milliseconds. The device is allowed to go idle for a few milliseconds if all of the selected process's requests have been satisfied, with the idea that the process may generate more requests within that window. If those requests don't come, that process's time slice ends. Later revisions of the patch check to see whether the given process is actually likely to run within the idle window, and preempt the slice immediately if the answer is "no." 

Jens [claims some very good results](<https://lwn.net/Articles/113869/>) for the new scheduler. The bandwidth numbers are nearly as good as those obtained with the anticipatory scheduler (AS), while the maximum latency is much less. These results may not be surprising; Jens has [borrowed code from AS](<https://lwn.net/Articles/114773/>), and the idle window has a similar effect to the brief I/O stalls used by AS to improve read bandwidth. As the I/O schedulers poach the best ideas from each other, they may well become more alike. The use of time slices may also improve the locality of accesses to the drive, reducing the amount of time lost to seeks. 

The new CFQ scheduler has spawned a low-key debate over which scheduler should be used by default. The default scheduler currently is AS, but some people ([Andrea Arcangeli in particular](<https://lwn.net/Articles/114774/>)) are saying that it should be CFQ instead. SUSE apparently already makes CFQ the default scheduler for its enterprise kernel. Andrew Morton is unsure; AS still seems to be better for desktop systems and IDE disks. Even so, he is [ready to consider a change](<https://lwn.net/Articles/114775/>) in the default scheduler: 

That being said, yeah, once we get the time-sliced-CFQ happening, it should probably be made the default, at least until AS gets fixed up. We need to run the numbers and settle on that. 

The AS scheduler has already seen one improvement: a fix for a bug that caused horrible performance for processes doing direct writes. Expect other changes as AS hacker Nick Piggin works at improving its performance. However this friendly competition turns out, better disk I/O performance for Linux users will be part of it.
