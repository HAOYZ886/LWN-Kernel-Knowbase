---
title: A potential NUMA scheduling solution
url: https://lwn.net/Articles/522093/
date: "October 31, 2012"
category: "Memory management-NUMA systems; NUMA; Scheduler-NUMA"
author: "By Jonathan Corbet October 31, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 31, 2012

Earlier this year, two different developers set out to create a solution to the problem of performance (or the lack thereof) on non-uniform memory access (NUMA) systems. The Linux kernel's scheduler will freely move processes around to maximize CPU utilization on large systems; unfortunately, on NUMA systems, that can lead to processes being separated from their memory, reducing performance considerably. Two very different solutions to the problem were posted, leaving no clear path toward a single solution that could be merged into the mainline. Now, perhaps, that single solution exists, but the way that solution came about raises some questions. 

The first approach was Peter Zijlstra's [sched/numa](<https://lwn.net/Articles/486858/>) patch set. It added a "lazy migration" mechanism (implemented by Lee Schermerhorn) that uses soft page faults to move useful pages to the NUMA node where they were actually being used. On top of that, it implemented a new "home node" concept that keeps the scheduler from moving processes between NUMA nodes whenever possible; it also tries to make memory allocations happen on the allocating process's home node. Finally, there was a pair of system calls allowing a process to change its home node and to form groups of processes that should all run on the same home node. 

Andrea Arcangeli's [AutoNUMA](<https://lwn.net/Articles/488709/>) patch set, instead, was more strongly focused on migrating pages to the nodes where they are actually being used. To that end, it created a tracking mechanism (again, using page faults) to figure out where page accesses were coming from; there was a new kernel thread to perform this tracking. Whenever the generated statistics revealed that too many pages were being accessed from remote nodes, the kernel would consider either relocating the processes performing those accesses or relocating the pages; either way, the goal was to get both the processes and the pages on the same node. 

To say that the two developers disagreed on the right solution is to understate the case considerably. Peter claimed that AutoNUMA abused the scheduler, added too much memory overhead, and slowed scheduling decisions unacceptably. Andrea responded that sched/numa would not work well, especially for larger jobs, without manual tweaking by developers and/or system administrators. The conversation was rather less than polite at times — until it went silent altogether. Peter last responded to the AutoNUMA discussion at the end of June — [this example](<https://lwn.net/Articles/522222/>) demonstrates the level of the discussion at that time — and [the last sched/numa posting](<https://lwn.net/Articles/508966/>) happened at the end of July. 

The silence ended on October 25 with Peter's posting of the [numa/core patch set](<https://lwn.net/Articles/521274/>). The patch introduction reads: 

Here's a re-post of the NUMA scheduling and migration improvement patches that we are working on. These include techniques from AutoNUMA and the sched/numa tree and form a unified basis - it has got all the bits that look good and mergeable.... 

These patches will continue their life in tip:numa/core and unless there are major showstoppers they are intended for the v3.8 merge window. We believe that they provide a solid basis for future work. 

It is worth noting that the value of "we" is not well defined anywhere in the patch set. 

Numa/core brings in much of the sched/numa patch set, including the lazy migration scheme, the memory policy changes, and the home node concept. The core scheduler change tries to keep processes on their home node by adding resistance to moving a process away from that node, and by trying to push misplaced processes back to the home node during load balancing. There is also a feature to wake sleeping processes on the home node regardless of where they were running before, but it is disabled because ""we found this to be far too aggressive"". Missing from this patch set is the proposed `numa_tbind()` and `numa_mbind()` system calls; it's not clear whether those are meant to be added later. 

The patch set also includes some ideas from AutoNUMA. The `page` structure gains a new `last_nid` field to record the ID of the NUMA node last observed to access the page. That new field will cause `struct page` to grow on 32-bit systems, which is never a popular thing to do. It is expected, though, that most systems where better NUMA scheduling really matters will be 64-bit. 

Scanning of memory is still done: pages are marked as being absent so that usage patterns can be observed from the resulting soft faults. But the kernel thread to perform this scanning no longer exists; it is, instead, done by each process in its own context. The number of pages scanned is proportional to each process's run time, so little effort is put into the scanning of pages belonging to processes that rarely run. Scanning does not start until a given process has accumulated at least one second of run time. It makes sense that there is little value in optimizing the NUMA placement of short-lived processes; in this case, that intuition was confirmed with an improvement in the all-important kernel-compilation benchmark. Most of the memory overhead added by the original AutoNUMA patches has been removed. 

Thus far, there has been little in the way of reviews of this large patch set, and no benchmark results posted. Things will have to pick up on that front if a patch set of this size is going to be ready by the time the 3.8 merge window opens. The numa/core patches may improve NUMA scheduling, and they may be the right basis to move forward with, but the development community as a whole does not know that yet. 

There is one other thing that jumps out at an attentive observer. These patches credit Andrea's work with a set of `Suggested-by` and `Based-on-idea-by` tags, but none of them are signed off by Andrea. It would appear that, while some of his ideas have found their way into this patch set, his code has not. But, despite the fact that he did not write this code, Andrea has been conspicuously absent from the review discussion. 

In the absence of any further information, it is hard not to conclude that Andrea has removed himself from this particular project. Certainly Red Hat cannot be faulted if it is unable to feel entirely comfortable when some of its highest-profile engineers are fighting among themselves in a public forum. So it is not hard to imagine that the developers involved were given clear instructions to resolve the situation. If that were the case, we would have a solution that was arrived at as much by Red Hat management as by the wider development community. 

Such speculation (and it certainly is no more than that), of course, says nothing about the quality of the current patch set. That will be judged by the development community, presumably between now and when the 3.8 merge window opens. Assuming the patches pass this review, we should have an improved NUMA scheduler and an end to an ongoing dispute. As the number of NUMA (and NUMA-like) systems grows, that can only be a good thing.
