---
title: Virtual machines as containers
url: https://lwn.net/Articles/684614/
date: "April 23, 2016"
category: Containers
author: "By Jonathan Corbet April 23, 2016 LSFMM 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
April 23, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

Containers and virtualization are two distinct mechanisms for sharing a physical host across multiple tenants. Containers tend to be more resource-efficient than virtualization, but virtual machines can provide stronger isolation. Rik van Riel started a memory-management track session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit by stating that there is an increasing level of interest in using virtual machines as if they were containers. One problem that results is that each virtual machine (VM) does its own caching, and, if left to its own devices, will fill its memory with cached data. That results in systems using much more memory than they really need, and reduces the number of VMs that can be packed into the host. 

A longstanding approach to this problem is [balloon drivers](<https://lwn.net/Articles/382299/>), which will "expand" by allocating memory from the guest and returning it to the host system. Ballooning is effective for extracting memory from guests, but it [![\[Rik van Riel\]](https://static.lwn.net/images/conf/2016/lsfmm/RikvanRiel-sm.jpg)](<https://lwn.net/Articles/684615/>) doesn't answer one important question: when should this be done? Despite years of experience with virtualization, we don't really know how to do this sort of memory balancing. 

James Bottomley suggested that it might be a good idea to use paravirtualization to move some memory-management decisions from the guest to the host. The [Clear Containers project](<https://lwn.net/Articles/644675/>), for example, is using the [DAX mechanism](<https://lwn.net/Articles/610174/>) — implemented to allow direct access to file data stored in persistent memory — to share file pages with the host. That works well, though sharing of anonymous pages would be harder. Perhaps the guest could share its LRU list with the host; the host could then see what the guest is trying to do and make more intelligent memory-balancing decisions. 

It should be possible to share all cached file data across the guests and the host if we had a paravirtualized page cache, James said: "how hard can it be?" 

Even if page caching is moved out of guests, though, there would still need to be a way to put memory pressure on guests. Other caches, such as the inode and dentry caches, could still expand to fill all available memory. So the need for a way to quantify memory pressure and communicate it between the host and the guests does not go away. As the session wound down, it was agreed that there were some interesting ideas in play. How soon those ideas will be turned into code remains to be seen, though.
