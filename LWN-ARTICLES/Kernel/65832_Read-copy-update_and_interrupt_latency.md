---
title: "Read-copy-update and interrupt latency"
url: https://lwn.net/Articles/65832/
date: "January 10, 2004"
category: "Read-copy-update"
author: 
---

The read-copy-update (RCU) algorithm has found many applications since it was added to the 2.5 kernel. By eliminating lock contention in many situations, RCU can greatly improve performance and scalability on multiprocessor systems. For more information on how RCU works, see [this description](<http://lwn.net/Articles/4974/>) or [this Driver Porting Series article](<http://lwn.net/Articles/37889/>). Or talk to the SCO Group, which claims to own any code which ever even dreamed of using RCU. 

It turns out, however, that there is one little problem with RCU - its effect on interrupt response times. RCU works by setting aside cleanup work until a later time, when it is known that the data structures of interest have no further references in the kernel. That cleanup work is done with a software interrupt, meaning it can happen after a hardware interrupt or at rescheduling time. But the list of RCU-protected data to be cleaned up can get quite long; it is used, for example, in high-turnover data structures like the dentry cache. So that software interrupt can, potentially, take a long time to run. The RCU cleanup code, in other words, can monopolize a processor for a relatively long period at just the times when a high-priority process might be trying to run. 

Dipankar Sarma has [taken a look at the situation](<https://lwn.net/Articles/65583/>) and found that processing RCU callbacks can, in some situations, take as much as 400 microseconds or so. That may not seem like a lot of time, but it can be enough to significantly increase response latencies. So he has sent out a set of patches which address the problem. 

In modern-day kernel programming, it sometimes seems like there is a standard answer to every problem: create a new kernel thread. Dipankar's patch does exactly that; it adds a new per-CPU "krcud" thread which handles RCU cleanup whenever the list of callbacks gets to be too long. Short callback lists are still dealt with at software interrupt time, since that is a faster way of doing things. But, if the list is too long (256 entries, by default) and, in particular, if there is a real-time process waiting to run, the tail end of the list is delegated over to krcud and control is returned to the scheduler. 

Dipankar reports good results in his tests, with overall system latencies of less than 400 microseconds. He's not pushing this patch for inclusion yet; it needs more testing first. But, if things pan out, a faster-responding 2.6 kernel may result in the near future.
