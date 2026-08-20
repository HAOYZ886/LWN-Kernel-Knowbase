---
title: The intersection of lazy RCU and memory reclaim
url: https://lwn.net/Articles/931920/
date: "May 18, 2023"
category: "Read-copy-update"
author: "By Jonathan Corbet May 18, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 18, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

Joel Fernandes introduced himself to the memory-management track at the 2023 [Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) as a co-maintainer of the read-copy-update (RCU) subsystem and an implementer of the "lazy RCU" functionality. Lazy RCU can improve performance, especially on systems that are not heavily utilized, but it also has some implications for memory management that he wanted to discuss with the group. 

The core idea behind lazy RCU is that, when the system is idle, it may not need to invoke RCU callbacks right away. These callbacks trickle in constantly, even on a lightly loaded system, and waking a CPU to call them can disturb an otherwise idle system, confusing the power-management code. This behavior can be seen in workloads like video playback on Chrome OS systems and Android logging. 

[![\[Joel Fernandes\]](https://static.lwn.net/images/conf/2023/lsfmm/JoelFernandes-sm.png)](<https://lwn.net/Articles/931922/>) RCU, he said, maintains a per-CPU "bypass list" to reduce contention. Normally, callbacks can be queued on one CPU but run on another, which can lead to lock contention and reduced performance. If the main callback list gets too long, the RCU code will start shunting callbacks over to the bypass list instead, avoiding the need to acquire a lock. Eventually the bypass list is flushed back onto the main list, either as the result of a timer firing or the bypass list getting too long. Lazy RCU is based on the idea that callbacks marked as non-urgent can go straight to the bypass list and be processed at some future time. This technique, he said, can reduce a system's power usage by 10-20%. 

One of the main uses of RCU callbacks is to release memory once it's safe to do so. Accumulating callbacks indefinitely thus has the potential to run the system short of memory over time. To avoid this problem, RCU implements a simple shrinker that flushes bypass-list callbacks into the main list, where they will be processed. There are some problems with this approach, though, starting with the fact that RCU has no way to know how much memory any given callback will release to the system. Shrinkers are for caches, but the callback list is not really a cache; it is, instead, a deferred garbage-collection mechanism. So a call to the shrinker might free more memory than is needed, but it doesn't do that immediately; instead, the shrinker has to trigger an RCU grace period, which can take some time. 

Fernandes was looking for input on how the handling of this list could be improved from a memory-management point of view. Michal Hocko said that the shrinker is probably not the right approach; the kernel's proactive-reclaim mechanisms can cause shrinkers to be called even when the system is not short of memory. That could cause callbacks to be flushed unnecessarily, defeating the purpose of lazy RCU. A better idea would be to hook into the allocator directly, perhaps in a function like [`should_reclaim_retry()`](<https://elixir.bootlin.com/linux/v6.4-rc1/source/mm/page_alloc.c#L4125>). When that call does happen, he said, RCU should just flush everything. Fernandes said this approach might help. 

Another attendee suggested that, since callbacks are being flushed in the hope that they will free memory, the right thing to do would be to specially mark callbacks that will do so? Matthew Wilcox said that "99% of RCU callbacks" free memory, but Fernandes disagreed, saying that they handle numerous other types of tasks as well. Still, he allowed that "most" callbacks do, indeed, free memory. Perhaps, he said, a better approach would be to create an API for callbacks that _don't_ return memory to the system. 

Fernandes asked whether it might make sense to get information from the memory-management subsystem on whether any given callback actually freed memory. That might help in cases where a specific amount of memory is targeted for freeing. He also wondered if the RCU shrinker should return zero (indicating no memory was actually freed), since it will have only started a grace period and will not have actually freed any memory yet. The answer was that RCU should just drop the shrinker and implement something better. 

There was a side discussion about [`kfree_rcu()`](<https://elixir.bootlin.com/linux/v6.4-rc1/source/include/linux/rcupdate.h#L958>), which exists for the sole purpose of freeing memory after an RCU grace period. This implementation, rather than maintaining a linked list of callbacks, just fills a page with pointers to the objects to be freed; the whole set can then be returned with a call to `kfree_bulk()`. This approach has a number of advantages, including increased cache locality and the ability to use the more efficient `kfree_bulk()` method. There is a significant disadvantage, though, in that `kfree_rcu()` may have to allocate memory while freeing. Having to allocate memory in this situation is something kernel developers go out of their way to avoid, since that allocation might be impossible at exactly the time when it is most needed. 

Fernandes would like to build a deferred-freeing mechanism directly into the slab allocator but, he confessed, he was "living in a fantasy world" when he was researching the idea. It is harder to do than he thought it would be. The interaction of grace periods with the slab allocator is tricky and, when the need to free memory arises, the grace period might have already passed, meaning that the RCU stage can be skipped entirely. 

His thought was to mark objects specifically in the slab allocators as not being ready to be freed quite yet. The allocators could maintain such objects in their free list, but not hand them out to new users until that marking goes away. That would eliminate the need to allocate memory in `kfree_rcu()`, and could eliminate the need for a separate shrinker as well. Unfortunately, the SLUB allocator maintains its free lists by storing pointers in the objects themselves — which is not advisable if the objects are still in use. Slab maintainer Vlastimil Babka said that he would think about the problem. 

Fernandes closed the session by saying that the benefits of this scheme may not justify the addition of more complexity to the slab allocator. For now, at least, hooking into the reclaim path as Hocko suggested is the direction this work seems likely to go.
