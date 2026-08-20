---
title: Mob rule for dentries
url: https://lwn.net/Articles/441438/
date: "May 4, 2011"
category: Containers; Dentry cache
author: "By Jonathan Corbet May 4, 2011"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 4, 2011

As was discussed at the [2011 filesystem, storage, and memory management summit](<https://lwn.net/Articles/436871/>), there is an increasing level of interest in restricting the amount of kernel memory which can be used by groups of processes. One area of special interest is the directory entry (dentry) cache; a malicious program can, by creating a deep enough directory hierarchy, run the kernel out of memory with an explosion of the size of the dentry cache. So limiting dentry use has some real appeal, especially for those working to ensure that containers running on a Linux system cannot interfere with each other. 

Pavel Emelyanov's [per-container dcache management patches](<https://lwn.net/Articles/441164/>) are a first attempt at limiting dentry use. This patch works by organizing dentries into "mobs," being groups of dentries all of which represent names in a specific subtree of the filesystem. If the root of a mob were the root of a container's filesystem namespace, all dentries created by that container would be contained within that mob. At that point, a simple sort of resource control can be applied: adding a dentry to a mob which has hit its maximum size would require the removal of another dentry to compensate. If no dentries an be removed, attempts to add others will fail. 

The patch set adds three new `ioctl()` calls: `FIMOBROOT` to create a new mob at a given point in the filesystem, `FIMOBSIZE` to set the maximum size of a mob, and `FIMOBSTAT` to query the current usage of a mob. Pavel is somewhat apologetic about this interface; he seems to think it will have to change before the work could be considered upstream. But the first step is get some discussion of the concept; so far, there have been no responses to Pavel's patches.
