---
title: O_HOT and O_COLD
url: https://lwn.net/Articles/494028/
date: "April 24, 2012"
category: Filesystems
author: "By Jonathan Corbet April 24, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 24, 2012

While storage devices are billed as being "random access" in nature, the truth of the matter is that operations to some parts of the device can be faster than operations to others. Rotating storage has a larger speed differential than flash, while hybrid devices may show a large difference indeed. Given that differences exist, it is natural to want to place more frequently-accessed data on the faster part of the device. But a recent proposal to allow applications to influence this placement has met with mixed reviews; the problem, it seems, is a bit more complicated than it appears. 

The [idea](<https://lwn.net/Articles/493981/>), as posted by Ted Ts'o, is to create a couple of new flags to be provided by applications at the time a file is created. A file expected to be accessed frequently would be created with `O_HOT`, while a file that will see traffic only rarely would be marked with `O_COLD`. It is assumed that the filesystem would, if possible, place `O_HOT` files in the fastest part of the underlying device. 

The implementation requires a change to the `create()` inode operation; a new parameter is added to allow the VFS layer to pass down the flags passed by the application. That change is the most intrusive part of the patch, requiring tweaks to most filesystems—43 files changed in all. The only filesystem actually implementing these flags at the outset is, naturally, ext4. In that implementation, `O_HOT` files will be placed in low-numbered blocks, while `O_COLD` files occupy the high-numbered blocks—but only if the filesystem is stored on a rotating device. Requesting `O_HOT` placement requires the `CAP_RESOURCE` privilege or the ability to dip into the reserved block pool. 

A lot of people seem to like the core idea, but there were a lot of questions about the specifics. What happens when the storage device is an array of rotating devices? Why assume that a file is all "hot" or all "cold"; some parts of a given file may be rather hotter than others. If an application is using both hot and cold files, will the (long) seeks between them reduce performance overall? What about files whose "hotness" varies over time? Should this concept be tied into the memory management subsystem's notion of hot and cold pages? And what do "hot" and "cold" really mean, anyway? 

With regard to the more general question, Ted [responded](<https://lwn.net/Articles/494033/>) that, while it would be possible to rigorously define the meanings of "hot" and "cold" in this context, it's not what he would prefer to do: 

The other approach is to leave things roughly undefined, and accept the fact that applications which use this will probably be specialized applications that are very much aware of what file system they are using, and just need to pass minimal hints to the application in a general way, and that's the approach I went with in this O_HOT/O_COLD proposal. 

In other words, this proposal seems well suited to the needs of, say, a large search engine company that is trying to get the most out of its massive array of compute nodes. That is certainly a valid use case, but a focus on that case may make it hard to generalize the feature for wider use. 

Generalizing the feature may also not be helped by placing the decision on who can mark files as "hot" at the individual filesystem level. That design could lead to different policies provided by different filesystems; indeed, Ted expects that to happen. Filesystem-level policy will allow for experimentation, but it will push the feature further into an area where it is only useful for specific applications where the developers have full control over the underlying system. One would not expect to see `O_HOT` showing up in random applications, since developers would have no real way to know what using that flag would do for them. And that, arguably, is just as well; otherwise, it would not be surprising to see the majority of files eventually designated as "hot." 

Interestingly, there is an alternative approach which was not discussed here. In 2010, a set of ["data temperature" patches](<https://lwn.net/Articles/398503/>) was posted to the btrfs list. This code watched accesses to files and determined, on the fly, which blocks were most in demand. The idea was that btrfs could then migrate the "hot" data to the faster parts of the storage device, improving overall performance. That work would appear to have stalled; no new versions of those patches have appeared for some time. But, for the general case, it would stand to reason that actual observations of access patterns would be likely to be more accurate than various developers' ideas of which files might be "hot." 

In summary, it seems that, while there is apparent value in the concept of preferential treatment for frequently-accessed data, figuring out how to request and implement that treatment will take some more time. Among other things, any sort of explicit marker (like `O_HOT`) will quickly become part of the kernel ABI, so it will be difficult to change once people start using it. So it is probably worthwhile to ponder for a while on how this feature can be suitably designed for the long haul, even if some hot files will have to languish in cold storage in the meantime.
