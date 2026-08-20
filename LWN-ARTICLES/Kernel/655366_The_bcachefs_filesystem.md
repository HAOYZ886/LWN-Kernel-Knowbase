---
title: The bcachefs filesystem
url: https://lwn.net/Articles/655366/
date: "August 25, 2015"
category: "Filesystems-bcachefs"
author: "By Jonathan Corbet August 25, 2015"
---

By **Jonathan Corbet**  
August 25, 2015

The Linux kernel does not lack for filesystem support; many dozens of filesystem implementations are available for one use case or another. But, after all these years, Linux arguably lacks an established "next-generation" filesystem with advanced features and a design suited to contemporary hardware. That situation holds despite the existence of a number of competitors for that title; Btrfs remains at the top of the list, but others, such as tux3 and (still!) reiser4, are out there as well. In each case, it has taken rather longer than expected for the code to reach the required level of maturity. The list of putative next-generation filesystems has just gotten longer with the recent [announcement](<https://lwn.net/Articles/655184/>) of the "bcachefs" filesystem. 

Bcachefs is an extension of bcache, which first [appeared](<https://lwn.net/Articles/394672/>) in LWN in 2010\. Bcache was designed as a caching layer that improves block I/O performance by using a fast solid-state drive as a cache for a (slower, larger) underlying storage device. Bcache has been steadily developed over the last five years; it was merged into the mainline kernel during the 3.10 development cycle in 2013. 

Mainline bcache is _not_ a filesystem; instead, it looks like a special kind of block device. It manages the movement of blocks of data between fast and slow storage, working to ensure that the most frequently used data is kept on the faster device. This task is complex; bcache must manage data in a way that yields high performance while ensuring that no data is ever lost, even in the face of an unclean shutdown. Even so, at its interface to the rest of the system, bcache looks like a simple block device: give it numbered blocks of data, and it will store (and retrieve) them. 

Users typically want something a bit higher-level than that; they want to be able to organize blocks into files, and files into directory hierarchies. That task is handled by a filesystem like ext4 or Btrfs. Thus, on current systems, bcache will be used in conjunction with a filesystem layer to provide a complete solution. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

It seems that, over time, bcache has developed the potential to provide filesystem functionality on its own. In the bcachefs announcement, Kent Overstreet said: 

Well, years ago (going back to when I was still at Google), I and the other people working on bcache realized that what we were working on was, almost by accident, a good chunk of the functionality of a full blown filesystem - and there was a really clean and elegant design to be had there if we took it and ran with it. 

The actual running with this idea appears to have happened relatively recently, with the first publicly visible version of the bcachefs code being [committed](<http://evilpiepirate.org/git/linux-bcache.git/commit/?h=bcache-dev&id=d16ce13a114e6d94dc2ddb01bee306be92315e32>) to the bcache repository in May 2015. Since then, it has seen a steady stream of commits from Kent; it was [announced](<https://lwn.net/Articles/655368/>) on the bcache mailing list in mid-July, and on linux-kernel just over a month later. 

With the bcachefs code added, bcache has gained the namespace and file-management features that, until now, had to be supplied by a separate filesystem layer. Like Btrfs, it is a copy-on-write filesystem, meaning that data is never overwritten. Instead, a block that is overwritten moves to a new location, with the older version persisting as long as any references to it remain. Copy-on-write works well on solid-state storage devices and makes a number of advanced features relatively easy to implement. 

Since the original bcache was a block-device management layer, bcachefs has some strong features in this area. Naturally, it offers multi-tier hybrid caching of data, and is able to integrate multiple physical devices into a single logical volume. Bcachefs does not appear to have any sort of higher-level RAID capability at this time, though; a basic replication mechanism is ""like 80% done"". Features like data checksumming and compression are supported. 

The plans for the future include filesystem features like snapshots — an important Btrfs feature that is not yet available in bcachefs. Kent listed [erasure coding](<https://en.wikipedia.org/wiki/Erasure_code>) as well, presumably as an alternative to higher-level RAID support. Native support for [shingled magnetic recording drives](<https://lwn.net/Articles/637035/>) is on the list, as is support for working with raw flash storage directly. 

But none of those features are present in bcachefs now; work has been focused on getting the basic filesystem working in a reliable manner. Performance tuning has not been a priority thus far, but the filesystem claims reasonable performance numbers already — though, as Kent admitted, it suffers from the common (to copy-on-write filesystems) problem of "filling up" well before the underlying storage is actually filled with data. Importantly, the on-disk filesystem format has not yet been finalized — a clear sign that a filesystem is not yet ready for real-world use. 

Another important (though unlisted) missing feature is a filesystem integrity checker ("fsck") utility. 

Bcachefs looks like a promising filesystem, even if many of the intended features have not yet been implemented. But those who have watched filesystem development for any period of time will know what comes next: a surprisingly long wait while the code matures to the point that it can actually be trusted for production workloads. This process, it seems, cannot be hurried beyond a certain point; that is why other next-generation filesystem efforts are seemingly never quite ready. The low-level device-management code in bcachefs is tested and production-quality, but the filesystem code lacks that pedigree. Kent says that it ""won't be done in a month (or a year)"", but the truth is that it may not be done for several years yet; that is how filesystem development tends to go. 

How many years depends, of course, on how many people test the filesystem and how much development effort it gets. Currently it has a development community of one — Kent — and he has noted that his full-time attention is ""only going to last as long as my interest and my savings account hold out"". If bcachefs acquires both a commercial sponsor and a wider development community, it may yet develop into that mature next-generation filesystem that we seem to never quite get (though Btrfs is there by some accounts). Until that happens, it should probably be looked at as an interesting idea with some advanced proof-of-concept code.
