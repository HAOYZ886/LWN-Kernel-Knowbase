---
title: "KS2008: Development tools"
url: https://lwn.net/Articles/298592/
date: "September 16, 2008"
category: Development tools; Patchwork
author: "By Jonathan Corbet September 16, 2008 LWN's 2008 Kernel Summit coverage"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 16, 2008

* * *

LWN's [2008 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2008/>)

Paul Mackerras was the leader of a kernel summit session dedicated to development tools. In the end, though, only one tool was discussed: the [Patchwork](<http://patchwork.ozlabs.org/>) system used by the PowerPC development community. Patchwork is a patch management system; its job is to ensure that posted patches are properly tracked, reviewed, and disposed of. 

The Patchwork system can be configured to watch a mailing list; whenever a message containing a patch is posted, it is added to the database. Any followup discussion is also captured and stored with the patch. Maintainers can go into the system, review patches, delegate them to other maintainers, and mark them for their final destination. Patches which are set to be merged into a subsystem tree can be grouped into bundles; the maintainer can then extract them as a mailbox file suitable for feeding to the `git-am` tool. 

A nice feature of Patchwork is that it can recognize messages containing `Acked-by` lines and automatically note the acks in the original patch. 

Patchwork was generally recognized as a useful tool; the developers began discussing whether it should be used for the kernel as a whole. It was noted that all maintainers need to commit to using it, or it will quickly clog up with patches that nobody is paying attention to. Nobody has any illusions that all kernel developers can be convinced to start working with this new tool; Andrew Morton stated that he was probably too stuck in his way to make use of it. Some alternatives - such as having patches automatically age out of the system - were discussed. But it was generally agreed that trying to deal with the full linux-kernel mailing list would probably be too big of a step at this time. 

So a more likely outcome is that one or more subsystems will start experimenting with Patchwork, perhaps running it on one of the kernel.org systems. The SCSI or ext4 subsystems may be the early adopters here. If that trial works out, expanding the use of Patchwork may be considered.
