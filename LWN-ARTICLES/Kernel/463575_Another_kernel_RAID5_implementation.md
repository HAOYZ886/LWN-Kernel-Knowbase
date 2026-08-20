---
title: Another kernel RAID5 implementation
url: https://lwn.net/Articles/463575/
date: "October 18, 2011"
category: "Block layer-RAID; RAID"
author: "By Jonathan Corbet October 18, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 18, 2011

There are many things that the kernel lacks, but RAID implementations is not on that list. Both the MD and DM subsystems currently have full RAID support, while the Btrfs filesystem has lower-level RAID support. [RAID5/6 support for Btrfs](<https://lwn.net/Articles/341026/>) has been posted a couple of times, but has not yet made it into the mainline. So, one might well be justified in wondering if yet another RAID5 implementation is needed in the kernel. 

There will be one if Boaz Harrosh has his way; his [RAID5 support patch](<https://lwn.net/Articles/463342/>) has been posted to a few filesystem-related kernel development lists. Boaz's patch is aimed at adding RAID5 support to the "objects raid engine" code in the exofs filesystem, which provides a POSIX filesystem on top of object-storage devices. It also implements RAID5 for the pNFS object-storage backend. 

According to Boaz, this work constitutes a nice, general-purpose RAID library that could be used in other settings; in particular, he says, Btrfs could make use of it. What would be even nicer, of course, is if some of the existing in-kernel RAID implementations could also move to this library - or if exofs could use one of those implementations. This version of RAID5 support may well be cleaner and more general than the others, but it may well take a stronger argument than that to get a new RAID subsystem merged at this point.
