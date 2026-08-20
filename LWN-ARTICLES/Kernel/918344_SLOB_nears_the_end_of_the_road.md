---
title: SLOB nears the end of the road
url: https://lwn.net/Articles/918344/
date: "December 23, 2022"
category: "Memory management-Slab allocators"
author: "By Jonathan Corbet December 23, 2022"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 23, 2022

The kernel project tries hard to avoid duplicating functionality within its code base; whenever possible, a single subsystem is made to serve all use cases. There is one notable exception to this rule, though: there are three object-level memory allocators ("slab allocators") in the kernel. The desire to reduce the count has been growing stronger over the years, and some steps have been taken in 6.2 to eliminate the least-loved allocator — SLOB — in the relatively near future. 

The job of a slab allocator is to provide the kernel with (usually) small chunks of memory in an efficient way. The heavily used [`kmalloc()`](<https://elixir.bootlin.com/linux/v6.1.1/source/include/linux/slab.h#L485>) function is implemented by the slab allocator, but there is also a lower-level API specialized for the management of numerous objects of the same size. It is quite common for a kernel subsystem to need to allocate instances of a given structure, for example; all of those instances are normally the same size and can be managed in a slab. 

The kernel's oldest slab allocator is typically just called SLAB (though the name is not an acronym); it has been there in one form or another since nearly the beginning. SLAB is intended to be a general-purpose allocator suitable for most workloads, and serves that purpose reasonably well. Even so, developers have occasionally wished for a different approach to object allocation; that led to the SLOB allocator, which was [added in 2006](<https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git/commit/?id=10cef602950>) for the 2.6.16 kernel release; its purpose is to support the smallest of systems where SLAB's memory use was seen as being too high: 

> SLOB is a traditional K&R/UNIX allocator with a SLAB emulation layer, similar to the original Linux kmalloc allocator that SLAB replaced. It's significantly smaller code and is more memory efficient. But like all similar allocators, it scales poorly and suffers from fragmentation more than SLAB, so it's only appropriate for small systems. 

The third allocator, SLUB, [followed](<https://git.kernel.org/pub/scm/linux/kernel/git/tglx/history.git/commit/?id=81819f0fc82>) for 2.6.22 in 2007. SLUB, which was intended to eventually replace SLAB, was a reaction to the perceived complexity and scalability problems in SLAB; it had a strong focus on scalability and performance. There have been attempts to add other allocators — [SLQB in 2008](<https://lwn.net/Articles/311502/>) and [SLEB](<https://lwn.net/Articles/388296/>) in 2010, for example — but the appetite for more allocators had mostly faded by then; Linus Torvalds [made it clear](<https://lwn.net/Articles/389402/>) in 2010 that he would not accept any more of them. So, in 2022, the set of mainline slab allocators remains SLAB, SLOB, and SLUB. 

Since then, there has been occasional talk of removing at least one of the existing slab allocators, but no real movement in that direction — until Vlastimil Babka took up the issue this year. In November, he [proposed](<https://lwn.net/ml/linux-kernel/b35c3f82-f67b-2103-7d82-7a7ba7521439@suse.cz/>) deprecating SLOB as a step toward its eventual removal: 

> The unsurprising reasons include code maintenance burden, other features compatible with only a subset of allocators (or more effort spent on the features), blocking API improvements (more on that below), and my inability to pronounce SLAB and SLUB in a properly distinguishable way, without resorting to spelling out the letters. 

The API improvement mentioned there is adding the ability to use [`kfree()`](<https://elixir.bootlin.com/linux/v6.1.1/source/mm/slab_common.c#L978>) on objects obtained from the lower-level slab functions (specifically [`kmem_cache_alloc()`](<https://elixir.bootlin.com/linux/v6.1.1/source/mm/slab.c#L3449>)); the SLOB allocator's version of `kfree()` can only handle objects allocated with `kmalloc()`, which complicates code elsewhere in the kernel. This cost is maybe justified if it brings other benefits, but Babka suggested that, in fact, nobody is actually using SLOB. The small devices it was aimed at ([less than 32MB of RAM](<https://lore.kernel.org/lkml/20051101210617.GS4367@waste.org/>)) don't exist in large numbers anymore, and even distributions for small devices (such as OpenWrt) are not using it. 

In the ensuing discussion, Paul Cercueil [described his attempts](<https://lwn.net/ml/linux-kernel/K5M3LR.3O50LSSSY5L3@crapouillou.net/>) to use SLOB, concluding that it simply does not work properly. Torvalds also [recalled](<https://lwn.net/ml/linux-kernel/CAHk-=wjBqwJk9RgfoBuM00T49M7uQiLnOvLST7L6v5rhS7fkDA@mail.gmail.com/>) hearing about SLOB problems in recent times; Aaro Koskinen [responded](<https://lwn.net/ml/linux-kernel/20221109234832.GE307514@darkstar.musicnaut.iki.fi/>), though, that SLOB might just be exposing driver bugs that are hidden by the other allocators. In the end, though, SLOB had no real defenders; nobody is advocating for keeping it in the kernel. 

Still, one cannot just remove an allocator from the kernel without trying to meet the needs of any existing users. So Babka posted [a patch set](<https://lwn.net/ml/linux-kernel/20221121171202.22080-1-vbabka@suse.cz/>) that tried to minimize the impact on any SLOB users that may remain. It introduces a new `SLUB_TINY` configuration option that removes many of the scalability and debugging features in an attempt to make SLUB as small as possible. SLUB will never be as small as SLOB but, hopefully, it can be made small enough for what passes as a "small system" in the 2020s. The `SLOB` configuration option was then [renamed](<https://lwn.net/ml/linux-kernel/20221121171202.22080-13-vbabka@suse.cz/>) to `SLOB_DEPRECATED`, and it gained a warning that SLOB would be removed ""in a few cycles"". This patch series was merged for 6.2. 

Unless a compelling case for the retention of SLOB comes up over the course of the next year, the removal of that allocator seems nearly certain. Then Linux will be down to just two allocators, both of which claim to handle the general case. 

Babka has made it clear that he sees two as still being too many; the removal of SLAB is sure to come up once the SLOB removal is complete. Getting rid of SLAB will not be quite so easy, though, since SLAB still has active users. Notably, Google is said to be using SLAB internally. It seems that SLUB is still worse for some use cases; as long as that situation persists, removing SLAB will be hard. This problem is complicated by a lack of general understanding of just where and why SLUB falls down. 

Getting a handle on that situation may take some time, so the kernel is likely to continue to have two slab allocators for some time yet. But even the removal of SLOB will make life easier in a number of ways. It can seem like code added to the kernel is there forever, but it is, sometimes, possible to get rid of old subsystems with enough patience and effort.
