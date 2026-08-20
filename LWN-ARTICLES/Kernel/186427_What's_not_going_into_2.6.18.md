---
title: "What's not going into 2.6.18"
url: https://lwn.net/Articles/186427/
date: "June 6, 2006"
category: Development model
author: 
---

The 2.6.17 development cycle is coming to an end, with the final release likely to happen before the middle of June. So, naturally, the attention of the kernel developers is turning toward the 2.6.18 cycle. As a way of encouraging thought on what should happen then, Andrew Morton has posted [a 2.6.18 merge plan summary](<http://lwn.net/Articles/186223/>) describing how he expects to dispose of the patches currently sitting in the -mm tree. There has been occasional talk of doing a bugfix-only kernel cycle, but it's clear that 2.6.18 won't be that cycle - there are a lot of patches tagged for merging. 

The features which are expected to be merged are interesting, but they are best discussed once they hit the mainline repository; until then, their fate remains uncertain. So, for now, suffice to say that 2.6.18 will likely include an S/390 hypervisor filesystem, a number of memory management patches, some software suspend improvements, a new i386 hardware clock subsystem, some SMP scheduler improvements, the [swap prefetch patches](<http://lwn.net/Articles/153353/>) (maybe), [priority-inheriting futexes](<http://lwn.net/Articles/178253/>), a rework of the `/proc/pid` code, a number of MD (RAID) improvements, a new kernel-space inotify API, and a bunch of code from subsystem trees which does not appear in -mm directly. As is usual, a great deal of code will be flowing into the mainline for the next release. 

It can also be interesting to look at what will _not_ be merged. From Andrew's posting, the following big patch sets are likely to be held back: 

  * There is a great deal of code which requires action by various subsystem maintainers. But, says Andrew, ""I continue to have some difficulty getting this material processed."" He will step up his efforts to get responses from maintainers, but some patches will likely continue to languish. 

In particular, some dismay has been expressed regarding how long it can take to get drivers into the mainline. It seems that, perhaps, the quality bar is being set too high. It is always possible to find things to criticize in a body of code, but sometimes the best thing to do is to proceed with the code one has and improve it as part of an ongoing process. There is concern that reviewers are insisting on perfection and keeping out code which is good enough, and which could be of value to Linux users. 

  * The [acx100 driver](<http://acx100.sourceforge.net/index.html>) supports a useful range of wireless chipsets. Unfortunately, there are some concerns about how this driver was developed and whether its inclusion could cause legal problems for Linux. Until that issue is resolved, this driver is likely to remain out in the cold. 

  * The per-task delay accounting patches are sitting on the edge. The main concern here appears to be that these patches create a new interface for getting per-task information from the kernel. Any other new code which exports that sort of information (and a number of patches exist) will be expected to use this new API. So more review and discussion may be called for here. There is also a separate patch set for non-task-oriented statistics which will probably not be merged this time around for the same reason. 

  * [eCryptfs](<http://lwn.net/Articles/156921/>) is uncertain as well. This filesystem implements its own mechanism for stacking on top of a base filesystem, but the primary reviewer would rather see the creation of a generic stacking layer for all to use. This is an issue which is often encountered by people trying to do new things; they are asked to make their infrastructure more generic. The intent is good, but it can cause delays and extra work for developers trying to add new features. 

  * The UTS namespaces patch. This patch, which implements a small part of the container concept, is not particularly useful on its own. So it will probably wait until more of the container infrastructure is in place. 

  * The [adaptive readahead patches](<http://lwn.net/Articles/155510/>) are deemed to be too young for now. Some benchmark results show significant performance improvements from these patches, but others are less clear. 

  * Reiser4. Says Andrew: ""We need to do something about this. It does need an intensive review and there aren't many people who have the experience to do that right, and there are fewer who have the time. Uptake by a vendor or two would be good."" This filesystem has been waiting on the sidelines for a very long time, and no prospective merge is yet in sight. 

  * The [generic IRQ code](<http://lwn.net/Articles/184750/>) is said to be "still stabilizing" and more likely to be merged in 2.6.19. That is also the case for the [lock validator](<http://lwn.net/Articles/185666/>). 

All of this is subject to change when the merge window actually opens. Developers are making cases for specific patches; Ingo Molnar is asking for reconsideration of the generic IRQ and lock validator patches, for example. Watch this space in the coming weeks to see what really happens.
