---
title: "Kernel Summit: Multipath I/O"
url: https://lwn.net/Articles/94569/
date: "July 21, 2004"
category: Device mapper; Multipath IO
author: 
---

This article is part of LWN's [2004 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2004/>).   
---  
A discussion of the device mapper and multipath I/O was led by Alasdair Kergon, the current DM maintainer. The device mapper crew is currently working on cluster code: clustered shapshots and mirroring, in particular. Things are going well, but they would like another hook into the block I/O system to help control how I/O requests are created for DM devices. After the discussion, it was not clear whether the block layer already provides what they need or not. 

The big area of interest is multipath I/O. In the past, the kernel has suffered from multiple, incompatible multipath implementations at various levels in the source. There is now a concerted effort afoot to concentrate multipath implementations on a new DM multipath target. Much of the code is there; it can handle multiple paths to devices, do simple load balancing on those paths, and quickly drop failing paths. At least, it quickly drops them once it is notified of the failure; there are still certain kinds of failure which can trigger long timeouts in the lower-level disk subsystems, leading to long delays before failing paths can be removed. 

In general, says Alasdair, things are "on track." There are still problems in getting cooperation from vendors, however. These problems come in two forms: access to information, and access to hardware. Multipath hardware is full of undocumented protocols and proprietary software; it is not always easy to get the information needed to write a proper free driver for it. It is also hard to get at the hardware itself; multipath storage systems tend to be very expensive and aren't something that most developers, or even their employers, can just run out and buy. So much multipath development is done using virtual devices or things like multi-port firewire disk drives. Much work can be done in that area, but it isn't quite what's needed to build credibility as an "enterprise class" multipath implementation. 

[>> Next: Virtualization](<https://lwn.net/Articles/94571/>).
