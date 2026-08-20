---
title: The NO_BOOTMEM patches
url: https://lwn.net/Articles/382559/
date: "April 7, 2010"
category: "Memory management-During early boot"
author: "By Jonathan Corbet April 7, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 7, 2010

Every kernel development cycle seems to involve one set of patches which turn out to be more trouble than had been expected. With 2.6.34, that award should probably go to the patches found under the somewhat confusing `CONFIG_NO_BOOTMEM` option. 

"Bootmem" is a simple, low-level memory allocator used by the kernel during the early parts of the bootstrap process. One might think that the kernel does not need yet another allocator, but the memory management code used during operation requires that much of the kernel already be functional before it can be called. Getting to that point involves a chain of increasingly complicated memory allocation mechanisms; on the x86 architecture, those begin the "early_res" mechanism which takes over from the BIOS "e820" facility. Once things get a little farther, the architecture-independent bootmem allocator takes over, followed, eventually, by the full buddy allocator. 

Yinghai Lu came to the conclusion that things could be simplified considerably if the bootmem stage were taken out of the picture. The result was a series of patches which extends the use of the early_res mechanism for long enough to bootstrap the buddy allocator. These changes were merged for 2.6.34, but the old bootmem-based code was left behind. The `CONFIG_NO_BOOTMEM` option controls which allocator is used, with the default being to short out bootmem. 

This is a significant change to the crucial and tricky early bootstrap code, so few people were surprised when some regressions were reported against 2.6.34-rc1. When the reports continued to arrive after -rc3, though, the level of irritation began to grow, to the point that Linus [started talking about](<https://lwn.net/Articles/382564/>) reverting the whole thing. Nobody seemed to dislike the objectives of the patches, but system-killer regressions after -rc3, along with the twisted mess of `#ifdef`s created by the patch and the fact that it was on by default led to some grumpiness. 

Normally, new features are expected to be configured out by default; to the greatest extent possible, a new kernel should behave as much like its predecessors as possible when the default options are taken. In this case, the default led to significant changes and problems. The purpose of this option [was twofold](<https://lwn.net/Articles/382566/>): to allow the new code to be configured out when it proved to be problematic, and to ensure that it was well tested in the mean time. Certainly it was successful on both fronts, even if some of the testers proved to be not entirely willing. 

As of this writing, it would appear that the worst problems have been fixed; talk of removing the no-bootmem code has subsided. Eventually, perhaps, all architectures will make similar changes and the bootmem code can be removed entirely. Meanwhile, Yinghai has a [new set of changes](<https://lwn.net/Articles/382571/>) on the horizon for 2.6.35: replacing the early_res code with the "logical memory block" allocator currently used by some other architectures. That change looks even more disruptive than the bootmem elimination was.
