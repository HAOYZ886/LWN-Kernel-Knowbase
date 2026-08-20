---
title: "LSFMM: Caching — dm-cache and bcache"
url: https://lwn.net/Articles/548348/
date: "May 1, 2013"
category: "Block layer-Caching"
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Two separate block-level caching solutions, dm-cache and bcache, were the topic of an LSFMM Summit 2013 discussion led by Mike Snitzer, Kent Overstreet, Alasdair Kergon, and Darrick Wong. Snitzer started things off with an overview of [dm-cache](<https://lwn.net/Articles/540996/>), which was included in the 3.9 kernel. It uses the kernel device mapper framework to implement a writeback or writethrough cache on a fast device for a slower "origin" device. 

Essentially, dm-cache uses the device mapper core and adds a policy layer on top of it. The policy layer is "almost like" a plugin interface, where different kinds of policies can be implemented, Snitzer said. Those policies (along with the cache contents) determine whether there is a hit or a miss on the cache and whether a migration (moving data between the origin and the cache device in either direction) is required. Various policies have been implemented, including least-recently used (LRU), most-frequently used (MFU), and so on, but only the default "mq" policy was merged, to reduce the number of policies being initially tested. 

There are hints that can be supplied by the filesystem to the policy, such as blocks that are dirty or have been discarded. That kind of information can help the policy make a more informed decision about where to store blocks. 

Overstreet then gave a status update for [bcache](<https://lwn.net/Articles/497024/>), which is queued for 3.10. There are "lots of users", he said, and the code has been relatively stable for a while. He has been concentrating mostly on bug fixes recently. Unlike dm-cache, which is "tiered storage", Overstreet said, bcache is more of a conventional cache. It can store arbitrary extents, down to a single sector, whereas with dm-cache, a block is either entirely cached or it isn't. 

Just before the summit, Wong sent an [email](<https://www.redhat.com/archives/dm-devel/2013-April/msg00068.html>) comparing the performance of bcache, dm-cache, and [EnhanceIO](<https://github.com/stec-inc/EnhanceIO>) to several mailing lists (dm-devel, linux-bcache, linux-kernel). He made a kernel that had each enabled and ran some tests. He found that EnhanceIO was the slowest, bcache had four to six times better performance, and dm-cache had better performance by a factor of 15, except when it didn't. All were compared to the same test being run on a regular hard disk, and sometimes, for reasons unknown, dm-cache performed more or less the same as the disk. He did note that some tests would cause the inode tables created by `mkfs` to be cached, which is not a particularly efficient use of the cache. 

Snitzer is trying to reproduce Wong's results, he said, but currently is getting poor results for both bcache and dm-cache. He said that he wants to get with Overstreet to try to figure it out. For his part, Overstreet cautioned against reading too much into synthetic benchmarks. They can be useful, but can also be misleading. Ric Wheeler asked if Snitzer was "seeing real improvements with real workloads"; Snitzer mentioned that switching between Git tags was one example where there was a clear win for dm-cache, but he needs some help in determining more real-world workloads. 

An attendee asked about whether the solutions always assumed the presence of a cache device. Snitzer said that dm-cache does make that assumption, but it is something that needs to change. Overstreet said that bcache does not require a cache device at all times. Since bcache is being used in production, it has had the time to hit the corner cases and handle situations where the cache device is unavailable. 

Snitzer said that there are still things that need to be done for dm-cache. Originally, it was doing I/O in parallel between the cache and the origin, but ultimately had to fall back to sequential I/O. Also, with [NVM devices](<https://lwn.net/Articles/547903/>) coming down the pipe, storage hierarchies are likely. Since dm is "all about stacking", dm-cache will fit well into that world, though Overstreet pointed out that bcache can stack as well. 

No real conclusions were reached, other than the need to get better "real-world" numbers for performance of both solutions. Figuring out why various testers are getting wildly different results is part of that as well.
