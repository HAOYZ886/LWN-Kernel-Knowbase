---
title: The sad story of the em28xx driver
url: https://lwn.net/Articles/306601/
date: "November 11, 2008"
category: "Development model-Driver merging"
author: "By Jonathan Corbet November 11, 2008"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 11, 2008

Over the last year or two, the kernel development process has been changed in a deliberate attempt to make the addition of new drivers easier. It has become clear that out-of-tree drivers often do not get any better until they are merged; meanwhile, users want those drivers and distributors are shipping them. So it would seem that everybody's interests are served by getting those drivers into the mainline tree. Experience with drivers merged under this policy has generally been positive; once those drivers head for the mainline, they get more attention and tend to improve quickly. 

Given that, one might well wonder why Markus Rechberger's recently submitted "empia" driver series is encountering so much resistance. This driver works with a number of video acquisition devices based on Empia chips; many of those are not supported by the kernel now. As an Empia Technology employee, Markus has access to the relevant data sheets and is, thus, well placed to write a fully-functional driver. There are users who will attest that the drivers work, and that Markus provides good support for them. But, as things stand now, it would appear that this driver is not headed for the mainline. 

What we have here is a classic story of an impedance mismatch between a developer and the development community. In the process, this long story has helped to give the Video4Linux development community a bit of a reputation as a dysfunctional family - a perception which those developers are only now beginning to overcome. The sad truth would seem to be that, while working with the community is something that a couple thousand developers do with little trouble every year, there will always be a few who have difficulties. 

A quick review of some of the history is in order here. Markus was one of the authors of the original em28xx driver, first merged for the 2.6.15 kernel. His efforts to enhance that driver quickly ran into trouble, though, when he tried to make substantial changes to the low-level tuner interface - changes which affected a number of other drivers. These changes were not popular in the Video4Linux community, and there were fears that they could break unrelated drivers. So this code was not merged. 

In response to this rejection, Markus [claimed ownership of the em28xx driver](<https://lwn.net/Articles/306603/>) and asked that it be removed from the mainline kernel. He then continued development of the code, hosting it on [his own server](<http://mcentral.de/wiki/index.php5/Main_Page>). There was even a period where the code was relicensed to the MPL, apparently as part of [an attempt to prevent it from being taken into the mainline](<https://lwn.net/Articles/306604/>). Eventually, Markus came back with [a new approach](<http://lwn.net/Articles/245527/>) which moved much of the tuner code into user space. That solution, too, failed to pass review; nobody else could really see much advantage in moving that much driver code out of the kernel. The fact that Markus clearly intended to have some of that code appear in the form of binary-only blobs did not help his case. So the user-space approach, like its predecessor, was not merged. 

While Markus was working on his own version of the code, others were putting patches into the mainline em28xx driver. At times, Markus tried to block those changes. The tone of the discussion is, perhaps, best seen from [this note](<https://lwn.net/Articles/306605/>) sent to Video4Linux maintainer Mauro Carvalho Chehab: 

Best would be to replace you as a maintainer since you don't have any respect of others work either. Companies should be aware that if they try to submit any code to you they will loose the authority over _their_ work. 

Of course, losing "authority" over code is inherent in releasing that code under a license like the GPL. This attempt to exercise control over freely-licensed code was [slapped down](<https://lwn.net/Articles/306606/>) by Andrew Morton and others, but it left unpleasant memories behind. 

Now Markus is back with a driver that, to all appearances, duplicates the functionality of a driver which is already in the mainline kernel. It is not hard to see this submission as an attempt to retake control of that driver and, perhaps, restart the discussions from past years. So it is not entirely surprising that this driver has not been received with a great deal of enthusiasm. In short, Markus has been told to go away until he is prepared to submit his work in the form of a series of small patches to the in-tree em28xx driver. 

The advantages of improving the current driver, rather than duplicating some of its functionality in a new code base, are clear. It would avoid the confusion which can come from having two drivers for the same hardware in the tree, and it would minimize the risk of losing important fixes which have been applied to the in-tree code. This is, also, the way that kernel developers are normally expected to do their work. On the other hand, video developer Hans Verkuil [reviewed the new driver](<https://lwn.net/Articles/306621/>) and concluded: 

In my opinion it's pretty much hopeless trying to convert the current em28xx driver into what you have. It's a huge amount of work that no one wants to do and (in this case) with very little benefit. 

This review notwithstanding, Mauro has [indicated](<https://lwn.net/Articles/306687/>) that he is not interested in accepting this patch. But rejecting Markus's new driver out of hand might just be a mistake. There seems to be little doubt that it has developed well beyond the in-tree driver; it supports a wider range of devices. Failure to merge it risks losing the work that has been done, and, perhaps, losing the future work of a developer who, for all his faults, is clearly trying to provide a better experience for Video4Linux users. 

Having multiple drivers for the same hardware in the kernel is not an ideal situation, but it is also not without precedent. The IDE and parallel ATA subsystems provide redundant support for a wide range of hardware. The e1000 and e1000e drivers had overlapping coverage for some time. In such cases, the long-term goal is usually to work toward the removal of one of the drivers. 

So one could make the case for merging the new driver and, eventually, removing the older one. In the process, the new driver could receive some much-needed attention from other developers. It has coding style and copyright attribution problems; a quick review has also left your editor wondering about locking issues. But such problems are common to drivers which have spent a lot of time out of tree; they are simply something to fix. Meanwhile, this driver contains the result of years of work and access to the relevant data sheets; freezing it out may not be in the best interests of kernel developers or users.
