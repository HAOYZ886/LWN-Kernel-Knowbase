---
title: "LCA: Rationalizing the wacom driver"
url: https://lwn.net/Articles/425870/
date: "February 1, 2011"
category: "Device drivers-Input"
author: "By Jonathan Corbet February 1, 2011"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
February 1, 2011

Wacom tablets are often the tool of choice for those who need accurate and flexible input devices; they seem to be especially favored by artists. Like a mouse, these tablets can report position and movement, but they can also present multiple input devices to the system (one for each of several different types of pens, for example) and report variables like pen angle, pressure, and more. Support in Linux for these devices has not been as good as one might like, but, as Peter Hutterer described in his talk at the linux.conf.au Libre Graphics Day miniconf, it is getting better quickly. How that came to be is a classic example of how to (or how _not_ to) manage kernel driver development. 

Peter is the maintainer for the bulk of the graphical input drivers. He has, he says, rewritten most of that subsystem, so he is to blame for the bugs which can be found there. Most input devices are easily handled through the evdev abstraction, but the Wacom driver is an exception. The things which are unique to these tablets (multiple input "devices," one associated with each pen, the pressure, tilt, and rotation axes, and the relatively high resolution) require a separate driver for their support. Thus, Wacom users must have the linuxwacom driver in their systems. 

There is some confusion about the linuxwacom driver, because there are multiple versions of it, all of which can be found on [SourceForge](<http://linuxwacom.sourceforge.net/>). One version (0.8.8) is created by Wacom itself; it is a classic vendor driver, Peter [![\[Peter Hutterer\]](https://static.lwn.net/images/conf/2011/lca/PeterHutterer-sm.jpg)](<https://lwn.net/Articles/425882/>) said, with everything that usually implies about the development process (code dumps) and the quality of the code itself. This driver ships as a tarball containing a wild set of permutations of kernel and X.org versions; it's a mess. But it's Wacom's mess, and the company has been resistant to efforts to clean it up. 

Peter got fed up with this situation in 2009 and forked the driver. His version is now the default driver in a number of distributions, and is the only one which supports newer versions of the X server. Looking at the repositories, Peter found 78 commits total before the fork, all from Wacom. After the fork, there are 788 commits, 65% from Red Hat, and 12% from Wacom. Extracting the driver from its vendor-dominated situation has definitely helped to increase its rate of development. 

Surprisingly, the original vendor driver is still under development by Wacom, despite the fact that it does not support current X servers and is not shipped by any distributors. The original mailing list is still in business, but, Peter warned, one should not ask questions about the new driver there. Kernel development, he said, should be done on the linux-kernel mailing list. There is also little point in talking to him about problems with the older driver; Wacom insists on keeping control over that code. 

**Update** : Peter tells us that there are three mailing lists (linuxwacom-announce, linuxwacom-discuss and linuxwacom-devel) which are still the place to go for general questions, including hardware-specific questions. X driver development for the forked driver happens exclusively on linuxwacom-devel and all patches are sent there. So the mailing lists are definitely the place to ask questions, at least in regards to the X driver. The kernel driver is the exception here. Kernel driver development should happen on LKML, not on linuxwacom lists. 

Much of the work Peter has done so far has been toward the goal of cleaning up the driver. That has involved throwing out a number of features. Some of those needed to go - the original driver tries to track the resolution of the screen, for example, which it has no business knowing. Support for the "twinview" approach to dual monitors has also been taken out. In some cases, the removed features are things that people want; support should eventually be restored once it can be done in the right way. Sometimes, Peter said, things have to get worse before they can get better. 

Also gone is the [wacomcpl](<http://linuxwacom.sourceforge.net/index.php/howto/wacomcpl>) configuration tool. It is, Peter said, some of the worst code that he has ever seen. 

Peter did this talk to update the graphics community on the state of support for this driver, but he was also looking for input. His attitude toward development was described as ""if it doesn't crash the server, it works"". In other words, he is not a graphic artist, so he has no deep understanding of how this hardware is used. To get that understanding, he needs input from the user community regarding development priorities and what does not work as well as it should. 

So artists making use of Wacom tablets should make sure that their needs are known; the developer in charge of the driver is ready to listen. Meanwhile, bringing a more open development process to the driver has increased the pace of development and is improving the quality of the code. If the usual pattern holds, before long Linux should have support for these tablets which is second to none.
