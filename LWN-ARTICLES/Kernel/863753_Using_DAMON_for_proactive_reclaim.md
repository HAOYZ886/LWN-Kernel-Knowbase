---
title: Using DAMON for proactive reclaim
url: https://lwn.net/Articles/863753/
date: "July 23, 2021"
category: "Memory management-DAMON"
author: "By Jonathan Corbet July 23, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
July 23, 2021

The DAMON patch set was first [covered here](<https://lwn.net/Articles/812707/>) in early 2020; this work, now in [its 34th revision](<https://lwn.net/ml/linux-kernel/20210716081449.22187-1-sj38.park@gmail.com/>), enables the efficient collection of information about memory-usage patterns on Linux systems. That data can then be used to influence the kernel's memory-management subsystem; one possible way to do that is to more aggressively reclaim memory that is not being used. To that end, DAMON author SeongJae Park is proposing [a DAMON-based mechanism](<https://lwn.net/ml/linux-kernel/20210720131309.22073-1-sj38.park@gmail.com/>) to perform user-controllable proactive reclaim. 

The core idea of DAMON is to use a sampling technique to determine which memory is in active use and which is sitting idle. A process's virtual address space is broken down into regions which vary in size depending on activity; the busiest regions are then subdivided over time for more fine-grained monitoring. Within each region, a randomly selected page is watched for activity, with the results being considered representative of the whole region. On demand, DAMON will produce a report in the form of a histogram informing the reader of how busy each memory region is. 

#### Putting DAMON to work

From the beginning, the motivation for this work has been to use DAMON data to improve system performance by tweaking the operation of the memory-management subsystem. The initial example is the [DAMON operation schemes (DAMOS) patch set](<https://lwn.net/ml/linux-kernel/20201216084404.23183-1-sjpark@amazon.com/>), which adds a mechanism that can make various [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) calls in response to usage patterns. Thus, for example, a request could be made to page out little-used memory, or to use huge pages for large, active regions. 

The proactive-reclaim patches build on DAMOS to take a more active role in memory management. This work adds a new kernel module that will monitor data out of DAMON, find any regions of memory that have not been accessed within a given time period, and actively reclaim the pages within those regions for other uses, even if the system is not feeling memory pressure at the moment. The results, seemingly, can be impressive, with significant improvements to the overall performance of the system in the form of fewer latency spikes and less time spent in direct reclaim. Getting there requires a number of changes, though. 

The first step is to add a "pageout" scheme to DAMOS, allowing the direct invocation of reclaim operations. This differs from the previously implemented `madvise()` mechanism in that it operates on physical pages, while `madvise()` works within the virtual address space of a single process. As suggested by its name, `madvise()` is advisory, while the new pageout scheme simply makes the reclaim happen. It thus will be far more effective at actually pushing pages out of memory. 

#### Making proactive reclaim efficient

It's worth noting that proactive reclaim is not a new idea; it was [discussed](<https://lwn.net/Articles/787611/>) at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, for example. At that time, the developers present agreed that there could be value in proactively reclaiming memory that appears to be unused; that is, after all, what the memory-management subsystem is meant to be doing in general. The problem is the cost; monitoring the system closely enough to figure out which pages are truly idle requires repeatedly scanning all of memory, which is expensive. That expense will, in many (if not most) cases, outweigh the benefits of accurate proactive reclamation of pages, so this work has not made its way into the mainline kernel. 

The whole idea behind DAMON is to make this kind of monitoring cheap; Park described it in the proactive-reclaim cover letter this way: 

> Its adaptive monitoring overhead control feature minimizes its monitoring overhead. It also let the upper-bound of the overhead be configurable by clients, regardless of the size of the monitoring target memory. While monitoring 70 GB memory of a production system every 5 milliseconds, it consumes less than 1% single CPU time. 

The granularity of the monitoring is configurable to get the overhead within whatever bounds the user finds acceptable; lower overhead, of course, leads to less accurate results. The efficiency of DAMON, it is hoped, can help to overcome the objections to the cost of proactive reclaim, but that is not a full solution to the problem; the reclaim operations themselves have a cost, and that cost cannot be allowed to run out of control. 

To avoid that problem, the proactive-reclaim patch set adds a number of knobs allowing the configuration of the new mechanism. One knob simply controls the amount of memory (expressed in bytes) that can be operated on in each (configurable) unit of time. Once that quota has been hit, proactive reclaim will take a proactive break until the next time period begins. To avoid getting stuck in a single memory region, proactive reclaim will move on to a new region in the next time slot if it runs into the limit. 

Beyond that, there is a knob setting a time quota, being the amount of CPU time that proactive reclaim is allowed to use per time period. Once again, when that time is exceeded, reclaim will pause until the next time slot opens up. Yet another knob allows the entire mechanism to be paused if the amount of free memory in the system exceeds a given watermark. There is, after all, little value to be had in using system resources to free memory when there is already more of it than is needed. Interestingly, proactive reclaim can also be paused if the amount of free memory is too _low_ ; in such situations, the memory-management subsystem will already be actively reclaiming memory and it is likely better for the proactive mechanism to just stay out of the way. 

Finally, there is a prioritization mechanism. Depending on the settings of the quota parameters, proactive reclaim may well be unable to work through all of memory in any reasonable time, so it makes sense to focus that work on the regions where the most benefit will be gained. Users can tweak yet another set of knobs to direct reclaim to specific regions based on their size, age, and access frequencies. 

#### Not for everybody

All of this work leads to the creation of a new mechanism with a lot of parameters to tweak and, probably, some real potential for harming the performance of the system rather than helping it. [This documentation patch](<https://lwn.net/ml/linux-kernel/20210720131309.22073-16-sj38.park@gmail.com/>) from the series describes the operation of this new module in detail. It seems unlikely to be a feature for casual users to play with. 

For administrators who need to get the best performance out of their systems and who have the time to tune it, though, proactive reclaim may offer some significant benefits. Benchmark results posted with the patch set show that an unconstrained reclaim module can free up nearly half of the available memory, but at a cost: nearly 12% of one CPU dedicated to the reclaim work, and a workload slowdown of just over 5% (presumably resulting from increased paging when reclaimed memory turns out to be needed after all). Applying some quotas can cut that overhead considerably while still gaining most of the free-memory benefit, which can be used to run other workloads. It seems likely, though, that the optimal settings will depend heavily on the nature of the workload. 

The proactive-reclaim work is in its third round on the mailing lists, but DAMON itself has now been posted 34 times, with no clear indication of when (or even if) it will be merged into the mainline. Park would clearly like to see some progress on that front; the cover letter begins with a request for reviews of the DAMON work. Those reviews — and an indication of wider interest in this work — will clearly be necessary if DAMON and the mechanisms built on it are to find their way into the mainline.
