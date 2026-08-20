---
title: Cache awareness for the CPU scheduler
url: https://lwn.net/Articles/1018334/
date: "April 29, 2025"
category: "Releases-7.2; Scheduler"
author: "By Jonathan Corbet April 29, 2025"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 29, 2025

The kernel's CPU scheduler has to balance a wide range of objectives. The tasks in the system must be scheduled fairly, with latency for any given task kept within bounds. All of the CPUs in the system should be kept busy if there is enough work to do, but unneeded CPUs should be shut down to reduce power consumption. A task should also run on the CPU that is most likely to have cached the memory that task is using. [This patch series](<https://lwn.net/ml/all/cover.1745199017.git.yu.c.chen@intel.com>) from Chen Yu aims to improve how the scheduler handles cache locality for multi-threaded processes. 

RAM is fast, but it is still unable to provide data at anything resembling the rate that a CPU can consume it. For this reason, systems are built with multiple layers of cache that are meant to hold frequently used data and make it available more quickly. Reading a value from cache is relatively fast; a read that goes all the way to RAM, instead, can stall a CPU for the time it takes to execute hundreds of instructions. Making effective use of cache is, thus, important for an application to perform well. Well-written applications are implemented with cache behavior in mind, but the kernel has a role to play as well. 

Each layer of cache is accessible by a different number of CPUs; the closest (L1) cache may be specific to a single CPU, while the subsequent (slower, but often larger) layers of cache will be shared by a group of CPUs. The last-level cache (LLC) farthest from the CPUs and, thus, the slowest, but it tends to be the largest and is shared by the largest number of CPUs. Moving a task from one CPU to another may move it away from the data it has built up in cache, hurting its performance. If a task is moved to another CPU in the same socket, much of its cached data may still be available in the lower-level caches; if it is moved to another NUMA node, it may have to start anew with an empty (from its point of view) cache. 

Because moving a task can hurt its performance, the CPU scheduler tries to avoid doing that when it can. That objective often runs into conflict with others, such as the need to balance the load across the system to make the best use of the available CPU resources. What the scheduler currently does not do, though, is to try to identify groups of tasks that might be sharing resources and, as a result, could benefit from sharing a single cache if they were scheduled together. Spreading those tasks across the system, instead, could lead to contention as they fight to keep data in their local caches. 

In March, Peter Zijlstra posted [an RFC patch](<https://lwn.net/ml/all/20250325120952.GJ36322@noisy.programming.kicks-ass.net/>) to explore improving this situation. It is based on the idea that, if a process has multiple threads, those threads are likely to be sharing memory and could benefit from running within the same cache domain. It adds some instrumentation to the (already large) [`mm_struct` structure](<https://elixir.bootlin.com/linux/v6.15-rc3/source/include/linux/mm_types.h#L898>) that describes an address space, including a per-CPU array that is used by the scheduler to keep track of how much time threads using that `mm_struct` spend on each CPU in the system. This data decays over time, so recent usage is more strongly represented than usage in the distant, forgotten past (a few tens of milliseconds ago, say). 

When the time comes to wake a thread that had been waiting for some event, the scheduler goes to that per-CPU array and determines which CPU has spent the most time executing threads from the same process. If the thread of interest has been running elsewhere, it will be moved to the selected CPU, where it will be closer to the other threads and, with luck, benefit from sharing cache space with them. As Zijlstra [noted](<https://lwn.net/ml/all/20250325184429.GA31413@noisy.programming.kicks-ass.net/>) at the time: ""This patch is not meant to be merged, it is meant for testing and development. We need to first make it actually improve workloads"". 

Chen then picked up this work and made a number of improvements to it. The original code would move a task to the hot CPU even if the task was already running within the same LLC domain, and thus already sharing the largest cache with that CPU. In this case, the movement will just slow that task down without much, if any, performance benefit, so task migration is inhibited in that case. 

The other problem that turned up might seem obvious from the description of how the original patch works. Threads are migrated to the hot CPU without regard to how busy that CPU already is. If the number of threads is large, that gathering may well overload the target cache domain, hurting performance overall. This problem was addressed by looking at the overall usage statistics generated by the scheduler's load-balancing algorithm to ensure that the process owning the thread in question is not overloading the target domain. Specifically, if the process is using more than 25% of the CPU time in that domain, or has more than 33% of the overall load there, then the scheduler will not move more threads there. 

That has improved the situation, but this work is still in a relatively early state. For example, it [can fight with the load balancer](<https://lwn.net/ml/all/2c45f6db1efef84c6c1ed514a8d24a9bc4a2ca4b.1745199017.git.yu.c.chen@intel.com>): 

> The aggregation of tasks will move tasks towards the preferred LLC pretty quickly during wake ups. However load balance will tend to move tasks away from the aggregated LLC. The two migrations are in the opposite directions and tend to bounce tasks between LLCs. 

CPU scheduling is driven by a lot of heuristics that can often come into conflict. So a patch series adding yet another heuristic ("concentrate a process's threads in a single cache domain") is sure to bring more surprising interactions that, perhaps, need to be addressed with even more heuristics. 

There is also a question that has not yet been asked here: is collecting a process's threads the best way to identify tasks that would benefit from sharing a cache? It may not be, but it has the advantage of actually being possible; detecting cache sharing in tasks without that kind of direct relationship could be difficult, if it is possible at all. In the end, the fate of this patch series will depend on whether it actually shows improvements on real workloads without causing regressions for others. That is a high bar to clear for a change like this. The kernel may well have more cache-aware scheduling in the future, but it seems like it may take a while yet before it is ready.
