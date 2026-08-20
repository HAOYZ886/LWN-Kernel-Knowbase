---
title: Lazytime hits a snag
url: https://lwn.net/Articles/634803/
date: "February 25, 2015"
category: "Filesystems-Access-time tracking"
author: "By Jonathan Corbet February 25, 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 25, 2015

The ["lazytime" concept](<https://lwn.net/Articles/621046/>) was first posted by Ted Ts'o in November 2014\. It attempts to address the performance costs of tracking the access time of each file while maintaining a more accurate notion of the last access time than the "relatime" option provides. In short, the last-access time is always kept current for as long as a file's inode is in memory; it is only written to persistent store if (1) there is another reason to write out the inode, or (2) the inode is being evicted from the cache. After a rewrite (to make it work at the virtual filesystem layer rather than being an ext4-specific option), lazytime was merged for the upcoming 4.0 kernel. 

That does not necessarily mean that 4.0 users will be able to enable this option, though. Jan Kara has [identified some problems](<https://lwn.net/Articles/634804/>) with the implementation that can cause incorrect times to be recorded in some situations. The issues look serious enough that use of lazytime in its current form is probably not a good idea. Ted [is looking into the report](<https://lwn.net/Articles/634805/>), noting that the option can be disabled before the 4.0 release if the problems are not easily fixed. 

[According to Jan](<https://lwn.net/Articles/634806/>), that chances are that an easy fix will not be within reach. So it may well be that, while the 4.0 kernel will have the lazytime code, users will not yet have access to it. Having kernel features work as intended before exposing them to users is one thing developers cannot be lazy about.
