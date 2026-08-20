---
title: Allocating uninitialized file blocks
url: https://lwn.net/Articles/492959/
date: "April 17, 2012"
category: fallocate
author: "By Jonathan Corbet April 17, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 17, 2012

The `fallocate()` system call can be used to increase the size of a file without actually writing to the new blocks. It is useful as a way to encourage the kernel to lay out the new blocks contiguously on disk, or just to ensure that sufficient space is available before beginning a complex operation. Filesystems implementing `fallocate()` take care to note that the new blocks have not actually been written; attempts to read those uninitialized blocks will normally just return zeroes. To do otherwise would be to risk disclosing information remaining in blocks recently freed from other files. 

For most users, `fallocate()` works just as it should. In some cases, though, the application in question does a lot of random writes scattered throughout the file. Writing to a small part of an uninitialized extent may force the filesystem to initialize a much larger range of blocks, slowing things down. But if the application knows where it has written in the file, and will thus never read from uninitialized parts of that file, it gains no benefit from this extra work. 

How much does this initialization cost? Zheng Liu recently implemented [a new `fallocate()` flag](<https://lwn.net/Articles/492920/>) (called `FALLOC_FL_NO_HIDE_STALE`) that marks new blocks as being initialized, even though the filesystem has not actually written them; these blocks, will thus contain random old data. A random-write benchmark that took 76 seconds on a mainline kernel ran in 18 seconds when this flag was used. Needless to say, that is a significant performance improvement; for that reason, Zheng has proposed that this flag be merged into the mainline. 

Such a feature has obvious security implications; Zheng's patch tries to address them by providing a sysctl knob to enable the new feature and defaulting it to "off." Still, Ric Wheeler [didn't like the idea](<https://lwn.net/Articles/492963/>), saying ""Sounds like we are proposing the introduction a huge security hole instead of addressing the performance issue head on"". Ted Ts'o was [a little more positive](<https://lwn.net/Articles/492964/>), especially if access to the feature required a capability like `CAP_SYS_RAWIO`. But everybody seems to agree that a good first step would be to figure out why performance is so bad in this situation and see if a proper fix can be made. If the performance issue can be made to go away without requiring application changes or possibly exposing sensitive data, everybody will be better off in the end.
