---
title: LIRC delurks
url: https://lwn.net/Articles/297770/
date: "September 10, 2008"
category: "Development model-Driver merging; Device drivers"
author: "By Jonathan Corbet September 10, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 10, 2008

The [Linux Infrared Remote Control project](<http://www.lirc.org/>) (LIRC) provides drivers for a number of infrared receivers and transmitters. It is, perhaps, most heavily used by people running MythTV and similar packages; it would, after all, completely ruin the experience to have to get up from the couch to change channels. Despite their established user base, and despite the fact that a number of distributors ship the code, the LIRC drivers have never found their way into the mainline kernel. In more recent times, little effort has gone into their development and maintenance; the link to "Caldera OpenLinux" on the project's web site would seem to make that clear. 

But LIRC is useful code, and, as is the case with most out-of-tree drivers, most people would really rather see LIRC in the mainline kernel. Merging into the mainline got a step closer on September 9, when Jarod Wilson [posted a version of the LIRC drivers](<http://lwn.net/Articles/297494/>) for consideration. Jarod, it seems, has been working (with Janne Grunau) on these drivers for some months; in the process, they have eliminated "tens of thousands" of complaints from the checkpatch.pl script and cleaned up a number of things. 

Even after that work, though, the LIRC drivers are clearly not yet up to normal kernel standards. Some very strange coding conventions are used in places. Many of the drivers have broken (or completely absent) locking. Duplicated code abounds. One driver has implemented a command parser in its `write()` function. Another driver is for hardware which already has a different driver in the mainline. And, importantly, these drivers do not work with the input subsystem. 

[PULL QUOTE:  The LIRC drivers would appear to strongly support the notion that out-of-tree code is, almost by necessity, worse code.  END QUOTE] In the past, Linus Torvalds (and others) have argued for merging drivers as soon as possible. If the code is poor, its chances of being improved get much higher once it's in the mainline and others can fix it. The LIRC drivers would appear to strongly support the notion that out-of-tree code is, almost by necessity, worse code. These drivers have been around for almost a decade, have been packaged by distributors, and have been used by large numbers of people. Despite all of that, they contain a large number of serious problems which have never been addressed. 

Now that the drivers have been posted to the linux-kernel list, quite a few of these problems are being pointed out; Jarod and Janne have been responding to reviews and fixing the issues. The "merge drivers early" philosophy would argue for pushing LIRC into 2.6.28, even if serious problems remain. Presence in the mainline will raise the visibility of the code, inspiring (one hopes) more developers to work on fixing it up. Merging LIRC will also free distributors from the need to create separate packages for those drivers. 

One important question will have to be addressed before merging LIRC can be seriously considered, though: its user-space API. Once LIRC is merged, its user-space API will be set in stone, so any problems with that API need to be resolved first. LIRC, being out of the mainline, did not follow the development of the input subsystem, so it does not behave like other input drivers - even in-tree drivers for infrared remotes. The use of an in-kernel command-line parser in at least one driver is sure to raise eyebrows; that sort of interaction should really be handled via `ioctl()` or sysfs. All told, it is hard to imagine this code being merged until the API problems have been resolved. 

Changing the LIRC API will, of course, lead to problems of its own. There is user-space code which depends on the current API; any changes will break that code. The kernel community will certainly understand this problem, but is unlikely to be swayed by it. There are a number of risks associated with maintaining production kernel code out of the mainline tree; one of those risks is that your established APIs will not be accepted by the kernel development community. So an API change may simply be part of the cost of getting LIRC into the mainline at this late date. 

It should be a cost worth paying. Once LIRC is in the mainline, interested developers will work to continue to bring the code up to kernel standards. The community will maintain it going forward. All Linux users will get the LIRC drivers with their kernel, with no need to deal with external packages. Getting there may be a bit frustrating for users of remotes and (especially) for the developers who have taken on the task of getting this code into the mainline. But, once it's done, remotes will just be more normal hardware, supported by the kernel like everything else.
