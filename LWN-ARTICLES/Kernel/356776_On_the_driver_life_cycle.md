---
title: On the driver life cycle
url: https://lwn.net/Articles/356776/
date: "October 13, 2009"
category: "Development model-Driver merging; Staging tree"
author: "By Jonathan Corbet October 13, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 13, 2009

As a general rule, all new features are supposed to be added to the kernel during the two-week merge window. There is an exception of sorts, though, for new device drivers. A well-written driver should not be able to cause regressions anywhere else in the kernel, and there is often value in getting it to users as quickly as possible. So drivers will often make it into the mainline when other large changes are barred. 

As the story of the recent [SCSI fixes pull request](<https://lwn.net/Articles/356778/>) shows, though, there are limits. This request included a pair of new drivers for high-end SCSI storage systems. Linus got grumpy for a few reasons: he would like to see subsystem maintainers try harder to get drivers in during the merge window, he thinks that the "driver exception" is mainly useful for consumer-level devices, and the driver in question here is not small bit of code - it's a 50,000 line monster. In the end, the driver was merged for 2.6.32-rc4, but Linus made it clear that he would rather see this kind of code during the merge window. 

The conversation drifted into whether the driver should have gone into the staging tree instead; those who looked at it did not describe it as the best code they had seen that day. SCSI maintainer James Bottomley [sees the staging tree](<https://lwn.net/Articles/356780/>) mainly as the place where user-space ABI issues are cleaned up. Mere code quality issues, he believes, are better handled directly in the SCSI tree. Others disagree; in the end, it will come down to what specific subsystem maintainers want to do. If the maintainer takes a new driver directly into the subsystem tree, nobody else can force it into staging instead. 

The discussion brought out another potential use for the staging tree - as a last resting place for old drivers on their way _out_ of the kernel. Staging maintainer Greg Kroah-Hartman [noted](<https://lwn.net/Articles/356783/>): 

It seems that I'm the only one that has the ability to drop drivers out of the kernel tree, which is a funny situation :) 

In thinking about this a lot more, I don't really mind it. If people want to push stuff out of "real" places in the kernel, into drivers/staging/ and give the original authors and maintainers notice about what is going on, _and_ provide a TODO file for what needs to happen to get the code back into the main portion of the kernel tree, then I'll be happy to help out with this and manage it. 

The idea remains hypothetical, though, until somebody actually uses the staging tree in this manner. It is hard to imagine a demotion to staging that would not be resisted by somebody; the first attempt to do so may well be interesting to watch.
