---
title: "Moving the -staging tree"
url: https://lwn.net/Articles/301192/
date: "October 1, 2008"
category: "Development model-Driver merging; Staging tree"
author: "By Jake Edge October 1, 2008"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
October 1, 2008

Greg Kroah-Hartman was tagged as the "maintainer of crap" at this year's [Kernel Summit](<http://lwn.net/Articles/298570/>) for his willingness to shepherd drivers of lower quality into the mainline. He has not shrunk from that label, when introducing a [patch set](<http://lwn.net/Articles/300890/>) that would merge some of those drivers. In fact, he has embraced the label: as part of his patch, he introduced the `TAINT_CRAP` flag for use in tainting kernels that load these, well, crappy drivers. 

There has been an [ongoing struggle](<http://lwn.net/Articles/270960/>) between those who want to see drivers get included as quickly as possible versus those who want to see them approach or attain normal kernel quality levels first. Kroah-Hartman started the [-staging tree](<http://lwn.net/Articles/285594/>) last June as a way to increase the visibility, thus testing and bug fixing, of out-of-tree drivers. Because drivers in that tree have been steadily improving--to the point where several have graduated to the mainline--the belief is that moving -staging itself into the mainline kernel will result in even faster progress. 

So, Kroah-Hartman has introduced a new directory (`drivers/staging`) to hold these drivers, as well as a mechanism to automatically taint the kernel if any of them get loaded. That will warn users when loading the module--at least if they check their logs--and include that info in any oops message that kernel might produce. Kernel hackers can then filter out problems depending on what the taint is--problems in kernels tainted with binary-only drivers are generally actively ignored. 

Getting those drivers into the mainline, though, will make it much easier for folks who want to test them. In addition, clean-ups and fixes for the drivers will go in as mainline patches, raising the visibility of the developers working on them. The change should have very minimal impact on other kernel users and developers. In particular, developers will not have to worry about reflecting API changes into `drivers/staging` as Kroah-Hartman will keep them up-to-date. 

The main [complaint](<https://lwn.net/Articles/301281/>) about the proposal has been that it duplicates the functionality or intent of the `EXPERIMENTAL` flag. There was also some belief that tainting the kernel was unduly harsh, but as Kroah-Hartman [points out](<https://lwn.net/Articles/301290/>): ""It isn't costing anything, and if a developer doesn't want to debug the kernel if such a driver is loaded, this allows them to do this."" 

As part of the thread, Paul Mundt [explains why `EXPERIMENTAL` has no meaning](<https://lwn.net/Articles/301293/>) in the kernel today: 

EXPERIMENTAL today is pretty damn meaningless. What it tends to mean in practice is that somethings needs some more testing, someone wants to be able to pull out the EXPERIMENTAL card when someone enables their option and their kernel blows up, the option/feature hasn't been around in the kernel for that long, or someone has just been too lazy to remove the flag (this last one probably covers about 90% of in-tree cases today). Stuff that is actively broken (in case of your kernel blowing up, not building, etc.) tends to be shoved under BROKEN instead. 

Mundt goes on to show the default configurations almost all enable `CONFIG_EXPERIMENTAL`, further reducing its meaning. It would be nice to audit all of the uses and restore the meaning of the flag, but that is beyond the scope of what Kroah-Hartman has set out to do. There still would be a difference, though, even if EXPERIMENTAL were meaningful. Mundt continues: 

The other key difference is that even with experimental stuff in the kernel, you will still get support, so it's not really a taintable offense. Stuff in staging/ on the other hand while potentially not actively hostile against the rest of the system, is still very much an unknown, and therefore the only safe thing to do is to taint the system and allow individual developers to make a choice regarding whether any resulting oopses are worth looking at or not. 

There are still some who are concerned about adding less-than-kernel-quality code. Randy Dunlap [puts it this way](<https://lwn.net/Articles/301307/>): ""I think that we have enough quality problems without adding crap."" But, Linus Torvalds has always been solidly in the "merge early" camp, so this proposal seems likely to go in for 2.6.28. Besides, as Stefan Richter [notes](<https://lwn.net/Articles/301309/>): 

OTOH many if not most of the -staging drivers are ones which are already in use. Their users already deal with whatever quality problems these drivers have, in addition to having to fight with the installation hassles that are inherent to out-of-tree drivers. 

In a fairly short span of time, merging drivers into the mainline has gotten a whole lot easier. At one time, developers might have to work on a driver for several development cycles before it reached a quality level that would allow it to be merged. In the interim, the -staging tree made things easier and more visible for testers and developers; soon that visibility will rise substantially again.
