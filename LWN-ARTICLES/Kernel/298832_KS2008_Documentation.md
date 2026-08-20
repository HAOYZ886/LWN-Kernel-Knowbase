---
title: "KS2008: Documentation"
url: https://lwn.net/Articles/298832/
date: "September 17, 2008"
category: Documentation
author: "By Jonathan Corbet September 17, 2008 LWN's 2008 Kernel Summit coverage"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 17, 2008

* * *

LWN's [2008 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2008/>)

Your editor got talked into kicking off the kernel summit discussion on documentation; if this coverage is sketchier than usual, it's because it's hard to try to lead a discussion and take notes at the same time. After some of the obligatory introductory notes on how documentation is always a problem, it was asked: how many kernel developers had actually gotten something useful from the in-tree documentation directory recently? Almost all attendees raised their hands. There is value, it seems, in the documentation which is available now. 

That said, there are also traps. An aspiring camera driver author would, upon exploring the documentation directory, stumble across a detailed file describing just how those drivers should be written. The author is Alan Cox, who might be considered to be a reasonably authoritative source. But this document describes the deprecated Video4Linux1 API; if our author wrote a new driver to that API, he or she would probably feel a little misled once the initial reviews came back. The value of that document in 2008 is probably negative. 

There are plenty of equally musty documents in the kernel documentation tree. The real problem is that documentation has no subsystem maintainer, nobody who will clean out the old stuff. The legendary lack of organization in that directory is also a result of a lack of overall maintenance. 

The question that was put to the developers was: what do you want from kernel documentation? Linus had a clear answer; what he wants is better release notes for each kernel version. It's not clear how to get there; maybe some sort of automated way of finding descriptions of new features in the git changelogs. What's even less clear is how this work could improve on the [high-quality work](<http://kernelnewbies.org/LinuxChanges>) done over at the kernelnewbies.org site. 

Matthew Wilcox asked for some quality control on documentation submissions. He noted, in particular, that the coding style document would appear to have drifted from its original intent over the years. 

One useful form of documentation that developers would like to see more of is test programs for new features. Test code for new system calls is especially useful; it describes how the system call should work, and allows architecture maintainers to verify that they have connected things up properly. 

There were questions on how much of the supplied kernel documentation is truly useful; maybe much of it should be removed? There are some obviously useful files, like those describing kernel boot and tuning parameters. The KernelDoc documents have their value; much of that documentation appears in the code itself, and the KernelDoc code checks to make sure that the documentation matches the associated function definitions. Much of the rest tends to be out of date and unused. 

One result of the discussion might be an effort to remove some of the oldest, most fictional documentation. Beyond that, though, it looks mostly like business as usual.
