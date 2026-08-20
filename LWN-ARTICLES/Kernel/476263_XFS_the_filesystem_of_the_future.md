---
title: "XFS: the filesystem of the future?"
url: https://lwn.net/Articles/476263/
date: "January 20, 2012"
category: "Filesystems-XFS"
author: "By Jonathan Corbet January 20, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 20, 2012

Linux has a lot of filesystems, but two of them (ext4 and btrfs) tend to get most of the attention. In his 2012 linux.conf.au talk, XFS developer Dave Chinner served notice that he thinks more users should be considering XFS. His talk covered work that has been done to resolve the biggest scalability problems in XFS and where he thinks things will go in the future. If he has his way, we will see a lot more XFS around in the coming years. 

XFS is often seen as the filesystem for people with massive amounts of data. It serves that role well, Dave said, and it has traditionally performed well for a lot of workloads. Where things have tended to fall down is in the ![\[benchmark plot\]](https://static.lwn.net/images/conf/2012/lca/xfs-before.png) writing of metadata; support for workloads that generate a lot of metadata writes has been a longstanding weak point for the filesystem. In short, metadata writes were slow, and did not really scale past even a single CPU. 

How slow? Dave put up some slides showing fs-mark results compared to ext4. XFS was significantly worse (as in half as fast) even on a single CPU; the situation just gets worse up to eight threads, after which ext4 hits a cliff and slows down as well. For I/O-heavy workloads with a lot of metadata changes - unpacking a tarball was given as an example - Dave said that ext4 could be 20-50 times faster than XFS. That is slow enough to indicate the presence of a real problem. 

#### Delayed logging

The problem turned out to be journal I/O; XFS was generating vast amounts of journal traffic in response to metadata changes. In the worst cases, almost all of the actual I/O traffic was for the journal - not the data the user was actually trying to write. Solving this problem took multiple attempts over years, one major algorithm change, and a lot of other significant optimizations and tweaks. One thing that was _not_ required was any sort of on-disk format change - though that may be in the works in the future for other reasons. 

Metadata-heavy workloads can end up changing the same directory block many times in a short period; each of those changes generates a record that must be written to the journal. That is the source of the huge journal traffic. The solution to the problem is simple in concept: delay the journal updates and combine changes to the same block into a single entry. Actually implementing this idea in a scalable way took a lot of work over some years, but it is now working; delayed logging will be the only XFS journaling mode supported in the 3.3 kernel. 

The actual delayed logging technique was mostly stolen from the ext3 filesystem. Since that algorithm is known to work, a lot less time was required to prove that it would work well for XFS as well. Along with its performance benefits, this change resulted in a net reduction in code. Those wanting details on how it works should find more than they ever wanted in [filesystems/xfs-delayed-logging.txt](<https://lwn.net/Articles/476267/>) in the kernel documentation tree. 

Delayed logging is the big change, but far from the only one. The log space reservation fast path is a very hot path in XFS; it is now lockless, though the slow path still requires a global lock at this point. The asynchronous metadata writeback code was creating badly scattered I/O, reducing performance considerably. Now metadata writeback is delayed and sorted prior to writing out. That means that the filesystem is, in Dave's words, doing the I/O scheduler's work. But the I/O scheduler works with a request queue that is typically limited to 128 entries while the XFS delayed metadata writeback queue can have many thousands of entries, so it makes sense to do the sorting in the filesystem prior to I/O submission. "Active log items" are a mechanism that improves the performance of the (large) sorted log item list by accumulating changes and applying them in batches. Metadata caching has also been moved out of the page cache, which had a tendency to reclaim pages at inopportune times. And so on. 

![\[benchmark plot\]](https://static.lwn.net/images/conf/2012/lca/xfs-after.png)

#### How the filesystems compare

So how does XFS scale now? For one or two threads, XFS is still slightly slower than ext4, but it scales linearly up to eight threads, while ext4 gets worse, and btrfs gets a lot worse. The scalability constraints for XFS are now to be found in the locking in the virtual filesystem layer core, not in the filesystem-specific code at all. Directory traversal is now faster for even one thread and much faster for eight. These are, he suggested, not the kind of results that the btrfs developers are likely to show people. 

The scalability of space allocation is "orders of magnitude" faster than ext4 offers now. That changes a bit with the ["bigalloc" feature](<https://lwn.net/Articles/469805/>) added in 3.2, which improves ext4 space allocation scalability by two orders of magnitude if a sufficiently large block size is used. Unfortunately, it also increases small-file space usage by about the same amount, to the point that 160GB are required to hold a kernel tree. Bigalloc does not play well with some other ext4 options and requires complex configuration questions to be answered by the administrator, who must think about how the filesystem will be used over its entire lifetime when the filesystem is created. Ext4, Dave said, is suffering from architectural deficiencies - using bitmaps for space tracking, in particular - that are typical of an 80's era filesystem. It simply cannot scale to truly large filesystems. 

Space allocation in Btrfs is even slower than with ext4. Dave said that the problem was primarily in the walking of the free space cache, which is CPU intensive currently. This is not an architectural problem in btrfs, so it should be fixable, but some optimization work will need to be done. 

#### The future of Linux filesystems

Where do things go from here? At this point, metadata performance and scalability in XFS can be considered to be a solved problem. The performance bottleneck is now in the VFS layer, so the next round of work will need to be done there. But the big challenge for the future is in the area of reliability; that may require some significant changes in the XFS filesystem. 

Reliability is not just a matter of not losing data - hopefully XFS is already good at that - it is really a scalability issue going forward. It just is not practical to take a petabyte-scale filesystem offline to run a filesystem check and repair tool; that work really needs to be done online in the future. That requires robust failure detection built into the filesystem so that metadata can be validated as correct on the fly. Some other filesystems are implementing validation of data as well, but that is considered to be beyond the scope of XFS; data validation, Dave said, is best done at either the storage array or the application levels. 

"Metadata validation" means making the metadata self describing to protect the filesystem against writes that are misdirected by the storage layer. Adding checksums is not sufficient - a checksum only proves that what is there is what was written. Properly self-describing metadata can detect blocks that were written in the wrong place and assist in the reassembly of a badly broken filesystem. It can also prevent the "reiserfs problem," where a filesystem repair tool is confused by stale metadata or metadata found in filesystem images stored in the filesystem being repaired. 

Making the metadata self-describing involves a lot of changes. Every metadata block will contain the UUID of the filesystem to which it belongs; there will also be block and inode numbers in each block so the filesystem can verify that the metadata came from the expected place. There will be checksums to detect corrupted metadata blocks and an owner identifier to associate metadata with its owning inode or directory. A reverse-mapping allocation tree will allow the filesystem to quickly identify the file to which any given block belongs. 

[![\[Dave Chinner\]](https://static.lwn.net/images/conf/2012/lca/DaveChinner-sm.jpg)](<https://lwn.net/Articles/476296/>) Needless to say, the current XFS on-disk format does not provide for the storage of all this extra data. That implies an on-disk format change. The plan, according to Dave, is to not provide any sort of forward or backward format compatibility; the format change will be a true flag day. This is being done to allow complete freedom in designing a new format that will serve XFS users for a long time. While the format is being changed to add the above-described reliability features, the developers will also add space for `d_type` in the directory structure, NFSv4 version counters, the inode creation time, and, probably, more. The maximum directory size, currently a mere 32GB, will also be increased. 

All this will enable a lot of nice things: proactive detection of filesystem corruption, the location and replacement of disconnected blocks, and better online filesystem repair. That means, Dave said, that XFS will remain the best filesystem for large-data applications under Linux for a long time. 

What are the implications of all this from a btrfs perspective? Btrfs, Dave said, is clearly not optimized for filesystems with metadata-heavy workloads; there are some serious scalability issues getting in the way. That is only to be expected for a filesystem at such an early stage of development. Some of these problems will take some time to overcome, and the possibility exists that some of them might not be solvable. On the other hand, the reliability features in btrfs are well developed and the filesystem is well placed to handle the storage capabilities expected in the coming few years. 

Ext4, instead, suffers from architectural scalability issues. According to Dave's results, it is not the fastest filesystem anymore. There are few plans for reliability improvements, and its on-disk format is showing its age. Ext4 will struggle to support the storage demands of the near future. 

Given that, Dave had a question of sorts to end his presentation with. Btrfs will, thanks to its features, soon replace ext4 as the default filesystem in many distributions. Meanwhile, ext4 is being outperformed by XFS on most workloads, including those where it was traditionally stronger. There are scalability problems that show up on even smaller server systems. It is "an aggregation of semi-finished projects" that do not always play well together; ext4, Dave said, is not as stable or well-tested as people think. So, he asked: why do we still need ext4? 

One assumes that ext4 developers would have a robust answer to that question, but none were present in the room. So this seems like a discussion that will have to be continued in another setting; it should be interesting to watch. 

[ Your editor would like to thank the linux.conf.au organizers for their assistance with his travel to the conference. ]
