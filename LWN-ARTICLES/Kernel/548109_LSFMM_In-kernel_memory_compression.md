---
title: "LSFMM: In-kernel memory compression"
url: https://lwn.net/Articles/548109/
date: "April 23, 2013"
category: Transcendent memory; zswap
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Compressed memory — the use of compression to compact memory contents in RAM as an alternative to swapping those contents to disk — has been the subject of extensive discussion on the kernel mailing lists. In particular, there has been a lot of debate over the relative merits of two approaches, known as zswap and zcache. Zswap developer Seth Jennings and zcache hacker Dan Magenheimer came to LSFMM 2013 in an attempt to reach an agreement on the way forward; an attempt that appears to have been successful. 

Seth started by presenting compressed swap and the problems that it is trying to solve. Readers who are not familiar with this concept may want to read [Seth's zswap article](<https://lwn.net/Articles/537422/>) and [this article by Dan](<https://lwn.net/Articles/545244/>) for the relevant background information. 

Andrew Morton noted that the whole thing makes his head spin. From there, the discussion quickly honed in on one of the more contentious sub-issues related to in-kernel memory compression: the need for a special-purpose memory allocator. There is a widespread sentiment among memory management developers that we have plenty of allocators already; adding another would increase maintenance costs significantly. In this case, though, there seems to be a legitimate need; in a sense, the "memory allocator" under discussion here is more of a bin-packing system that stores compressed pages of memory. The slab allocators are very good at packing fixed-size allocations, but compressed pages vary widely in size, so the existing allocators do not work for in-kernel compression. 

Agreement on the need for a new allocator is not the same as agreement on _which_ allocator should be used, though. Zswap uses "zsmalloc," which uses some clever schemes to pack compressed pages efficiently. That efficiency comes at a cost, though: freeing a page used by zsmalloc is difficult; it can require pushing an unknown number of pages out to the real swap device and an unknown amount of time. The "zbud" allocator used with zcache, instead, is inefficient: it can only pack two compressed pages into a physical page, but the cost of freeing that physical page is small and known ahead of time. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Hugh Dickins questioned the design of zbud, observing that it was strange to design that kind of limitation into the system from the start. But Mel Gorman came out strongly in favor of zbud despite its "awful packing properties." He sees the unpredictability at the heart of zsmalloc as a long-term source of bugs and strange memory management behavior. What he would really like, he said, would be a modular interface to the allocation layer so that different solutions could be used at different sites. 

From there the developers worked toward a consensus on how to handle the conflict between these two projects. There was some concern raised about the complexity of zcache, though the extra features provided by that complexity (primarily the ability to store compressed page cache pages) were appreciated. One of the data structures used by zcache was described in the session as "a table of hash tables of red-black trees of radix trees." So it is not surprising that the relative simplicity of zswap looked appealing. As Mel put it, if zswap cannot be made to work well, zcache is hopeless, so perhaps the best course is to start with zswap, possibly with a modular allocator interface. 

Hugh added that compression of page cache (file) pages may be appealing, but the filesystem developers do not seem to be that interested in zcache in general. So he agreed that it might make better sense to start with zswap, perhaps adding zcache features over time. Dan said that he would agree to merging zswap as long as there was an explicit understanding that zswap is not the end of development in this area; there is, he said, a lot more work to be done to gain the full benefits of in-kernel compression. In other words, he would plan to submit patches to increase the functionality of zswap over time. 

There was further discussion on various details, including writeback (the process by which compressed pages are uncompressed and written to the "real" swap device). Zswap does it in the zsmalloc allocator, which is seen as being the wrong place; the separate thread used by zcache looked better to some developers and was suggested as being a good first feature to port over to zswap. Hugh complained that writeback decisions should be made at a higher level altogether, though. 

Michel Lespinasse said that zswap, using the zsmalloc allocator, would work well in Google. They don't run with "real" swap at all, so the problematic writeback behavior associated with zsmalloc would not be experienced there. Zswap is not designed to run in this mode — it expects there to be a swap device to use as a backing store — but Mel suggested creating a fake swap device that would fail all requests as a possible solution there. 

In the end, it appears that there is a consensus for merging zswap as the next step for in-kernel compression. Mel noted that he would block the merging, though, if it didn't have a modular allocation layer. Leaving modularization for later would not work, he said; that work would never be done and he'd have to deal with the bug reports a couple of years down the line. So the allocation layer in zswap will need some work; after that, we will likely see a submission for mainline merging.
