---
title: Making sense of GFP_TEMPORARY
url: https://lwn.net/Articles/713076/
date: "February 1, 2017"
category: "Memory management-GFP flags"
author: "By Jonathan Corbet February 1, 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 1, 2017

This is the season where potential topics for the upcoming [Linux Storage, Filesystem, and Memory Management Summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit>) are discussed; often that discussion resolves the relevant issues before the actual event. That would appear to be the case with the mysterious `GFP_TEMPORARY` memory-allocation flag. The development community now know whats it does, but now it seems that the flag itself may turn out to be a temporary thing. 

Matthew Wilcox [started the discussion](<https://lwn.net/Articles/713152/>) by listing no less than nine different topics that he would like to see addressed at the summit. One of those (#8) was ""nailing down exactly what GFP_TEMPORARY means"". This flag was [added to the 2.6.24 kernel](<https://git.kernel.org/linus/e12ba74d8ff3e2f73a583500d7095e406df4d093>) by Mel Gorman in 2007; since then, it has picked up a few dozen users throughout the kernel. But, it seems, nobody has ever documented what the flag's effects are or when it should be used. 

What the flag actually does is relatively straightforward, though it took a while for the discussion to make it clear. Vlastimil Babka [described it](<https://lwn.net/Articles/713156/>) this way: 

GFP_TEMPORARY, compared to GFP_KERNEL, adds __GFP_RECLAIMABLE, which tries to place the allocation within MIGRATE_RECLAIMABLE pageblocks - GFP_KERNEL implies MIGRATE_UNMOVABLE pageblocks, and userspace allocations are typically MIGRATE_MOVABLE. 

All of this is driven by the need to [defragment memory](<https://lwn.net/Articles/368869/>) so that multiple-page allocations can be made when needed. Pages that are allocated for user-space memory are relatively easy to manage in this regard since they can be relocated elsewhere in physical memory; as long as the page-table entries are updated accordingly, the application(s) using those pages won't even be aware that they have moved. The kernel groups such pages together into regions of memory marked `MIGRATE_MOVABLE` in the hopes of being able to clear large contiguous areas of memory when the need arises. Keeping non-movable pages out of that area minimizes the risk of a single nailed-down page thwarting an effort to clear a range of memory. 

Memory allocated for the kernel is not so easy to relocate, though, since the memory-management subsystem has no way to know where the references to any given page of memory might be or even how many of them exist. Thus, as a general rule, kernel-space memory allocations must be assumed to be eternally fixed in place; they cannot be put into the `MIGRATE_MOVABLE` regions. That said, some kernel-space memory has at least the possibility of being freed when memory gets tight. Memory allocated from a slab allocator with an associated shrinker callback falls into this category, for example. If this "reclaimable" memory is grouped together and kept separate from the completely unmovable memory, then there is at least a chance of freeing some usable blocks of pages when the shrinkers are run. The `__GFP_RECLAIMABLE` flag indicates memory that can (maybe) be reclaimed by the kernel in this way. 

So `GFP_TEMPORARY` sets the `__GFP_RECLAIMABLE` flag, causing allocations to be taken from the `MIGRATE_RECLAIMABLE` block. That describes what the flag does, but not how it is meant to be used. After some discussion, it became evident that, in fact, nobody really seemed to know what the intended use case for `GFP_TEMPORARY` is. 

For example, one might imagine that, from its name, `GFP_TEMPORARY` is intended for short-lived allocations — those that will be freed in the near future. But, [Wilcox asked](<https://lwn.net/Articles/713162/>), what does short-lived mean in this context? Is it permissible for kernel code to block while holding a `GFP_TEMPORARY` allocation, for example? Or, instead, should preemption be disabled while holding that allocation? Would allocating data structures for I/O operations (which could take 30 seconds to time out) as `GFP_TEMPORARY` be acceptable? In other words, what are the promises that a kernel developer needs to make to perform a `GFP_TEMPORARY` allocation, and what benefits come from making those promises? 

With regard to the acceptable time period, it turns out there is no clear answer. In the above-linked message, Babka said: ""There's no simple connection to time, it depends on the larger picture"". This led to complaints from developers like [Neil Brown](<https://lwn.net/Articles/713165/>), who, understandably, thought that a name involving "temporary" would be somehow related to time. He also suggested that the whole idea is somewhat shaky, and that, if it works at all to reduce fragmentation, that is more a matter of luck. His suggestion was to look, instead, at mechanisms to render kernel-allocated objects movable so that active defragmentation could be performed. This is an interesting idea, but it is also less than trivial to implement and beyond the scope of the current discussion. 

Wilcox, meanwhile, [continued trying](<https://lwn.net/Articles/713166/>) to determine the situations in which `GFP_TEMPORARY` should be used. It seems that it should not be used with `kmalloc()` calls, since the slab allocators ignore it. It is possible to hold these allocations for a considerable period of time. He suggested that there might be two possible benefits from using `GFP_TEMPORARY`: a higher chance of successfully allocating the memory, and making larger allocations more likely to succeed in general. Babka [responded](<https://lwn.net/Articles/713167/>) that nothing in the memory-management code makes `GFP_TEMPORARY` allocations more likely to succeed, but that the general benefit for larger allocations might exist. 

In the end, nobody was able to come up with a simple answer to the question of when `GFP_TEMPORARY` should be used. So Michal Hocko [concluded](<https://lwn.net/Articles/713168/>) that perhaps it shouldn't exist at all: 

From the current discussion so far it really seems that it would be really hard to define sensible semantic for GFP_TEMPORARY with the current implementation so I will send a patch to simply drop this flag. If we want to have such a flag then we should start over with defining the semantic first and think this thing over properly. 

Subsystems like memory management are full of heuristics intended to improve the behavior of the system. The nature of heuristics, though, tends to make their use and benefits a bit fuzzy at times, especially in the absence of focused testing (as appears to be the case here). But even ineffective heuristics can end up wired into the system to the point where nobody questions their existence. One of the good things about free-software development is that it makes it easy for fresh eyes to come in and generate awkward questions.
