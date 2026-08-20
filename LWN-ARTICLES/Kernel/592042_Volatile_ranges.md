---
title: Volatile ranges
url: https://lwn.net/Articles/592042/
date: "March 26, 2014"
category: Volatile ranges
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

"Volatile ranges" are special regions of memory containing data that the owner application can regenerate if need be. If the system runs short of memory, the kernel is free to evict data from a volatile range, but otherwise the space is usable for activities like caching. The volatile range concept was raised again at the 2014 Linux Storage, Filesystem, and Memory Management Summit, in two separate sessions. This article is a combined look at both discussions. 

The first session started with an overview of the latest incarnation of the volatile ranges and `MADV_FREE` APIs; see [this article](<https://lwn.net/Articles/590991/>) for an overview of those proposals. One question that came up repeatedly concerned the need for a separate `vrange()` system call for volatile ranges. Some of the incarnations of that work have used `madvise()` instead, and some developers think that is the better approach. It turns out that one of the biggest arguments against an `madvise()` interface has to do with the process of marking pages as no longer being volatile. In that case, the system call needs to return two separate values: (1) how much memory was successfully marked non-volatile, and (2) whether any pages were purged by the kernel while they were marked as volatile. `madvise()` only allows for one return value, so it cannot be used to create that kind of interface. 

Should the interface indicate _which_ pages have been purged when a range is marked non-volatile? The current code returns a single boolean value indicating only whether any pages have been purged at all. Hugh Dickins said that some users would like to have more detailed information. That said, there does not appear to be any plan to extend the interface in that direction at this point. 

Another question has to do with page aging. When pages are marked as being volatile, should they be "aged" to look like they have not been referenced for a long time? Aging the pages in that way would cause them to be among the first that are reclaimed if the system encounters memory pressure. There does not seem to be much consensus on whether this kind of aging should be performed; if it is added, it might be under the control of a separate flag allowing user space to select the behavior it wants. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Hugh said he didn't like the `vrange()` name; he would rather see the name be a verb describing the action that is to be performed. There was also talk of making an `madvise2()` system call that would be able to provide the needed API. In the end, though, suggestions for better names have been in short supply, and Hugh agreed that, given all the revisions that volatile ranges have been through, keeping that functionality as a separate system call might be the best approach to take. 

Keith Packard raised a related use case that he has: graphics drivers can allocate large amounts of memory for caching that they can give up if need be. But the existing shrinker interface is not actually invoked often by the kernel, so he ends up holding memory rather longer than is warranted. Perhaps the volatile range functionality could be made available in a form that could be used by drivers as well? 

A couple of other API issues came up toward the end of the session. One had to do with what happens if a process writes to memory that is in a volatile range: in that case, should the memory remain volatile, or should writing the memory automatically make it non-volatile? Some developers would like to see the latter behavior, but John Stultz, the author of current versions of the patch, is uncomfortable with changing the state of pages on writes in that way. 

The current interface is memory-based, in that a volatile range is described by a base address and a length. Some versions of the patch have, instead, used a file-based interface, where a volatile range is described as a portion of a file. The Android "ashmem" subsystem, which, it is hoped, can be replaced by volatile ranges someday, uses a file-based interface, but John said that it could be changed internally to use a memory-based method instead. Keith had a bit of a stronger requirement for a file-based interface, though. The graphics system, he said, does not normally have addresses for most of the memory it uses for caching, and mapping all of that memory could create problems on 32-bit systems where there is not a lot of address space available. So he would rather see a file-based API. 

In the end, there was little in the way of concrete conclusions from this session. There will certainly be another version of the volatile ranges patch set at some point, but what it will look like is not entirely clear. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
