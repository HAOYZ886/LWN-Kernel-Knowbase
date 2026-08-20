---
title: The real BKL end game
url: https://lwn.net/Articles/424657/
date: "January 26, 2011"
category: Big kernel lock
author: "By Jonathan Corbet January 26, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
January 26, 2011

The removal of the big kernel lock (BKL) has been on the kernel community's "to do" list almost since that lock was first added to make the kernel work on multiprocessor systems. Over time, the significance of the lock has diminished as finer-grained locking was added to various kernel subsystems, but the BKL itself has endured. Getting rid of it for good remained desirable because the BKL can still cause unwanted latencies at times. There's also a certain amount of pride involved in completing the job. That completion has been long in coming, though; once the worst performance issues associated with the BKL were resolved, interest in doing the low-level work needed to finish the job declined. 

Two years or so ago, though, developers started working on BKL removal again. Some of this work was motivated by the realtime tree, where patience with latency sources is rather more limited. Still, it seemed like completion remained a distant goal; hundreds of BKL call sites remained in the kernel. 

Then Arnd Bergmann took on the task of eliminating the BKL entirely. His cleanup work has been going on for some time; if he has his way, [this patch set](<https://lwn.net/Articles/424637/>) (or something derived from it) will remove the BKL entirely in 2.6.39. To get there, about a dozen modules need to be addressed. Some of them (i830, autofs3, and smbfs) are simply to be removed. Others (appletalk and hpfs) are to be moved to the staging tree for near-term removal, though there is some resistance to that idea. The remaining modules are to be fixed in some way. Once that's taken care of, [the final patch in the series](<https://lwn.net/Articles/424677/>) removes the lock itself. It will not be missed.
