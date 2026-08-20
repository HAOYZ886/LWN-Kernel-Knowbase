---
title: "Kernel Summit 2007 - an advance view"
url: https://lwn.net/Articles/246565/
date: "August 24, 2007"
category: Kernel Summit
author: "By Jonathan Corbet August 24, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 24, 2007

For the past several years, the annual, invitation-only kernel developers' summit has been held immediately prior to the Ottawa Linux Symposium. This year is different, though: the summit is, instead, happening just after [LinuxConf Europe](<http://www.linuxconf.eu/2007/index.shtml>) in Cambridge, UK. As usual, your editor will be there and will be able to report from the event. The [preliminary agenda](<http://thunker.thunk.org/pipermail/ksummit-2007-discuss/2007-August/000398.html>) has been posted, though, as has [the list of attendees [PDF]](<http://thunker.thunk.org/pipermail/ksummit-2007-discuss/attachments/20070823/da665b0c/ks2007-invites-v2-0001.pdf>). So it is possible to look forward and get a sense for what is likely to be discussed. 

A few months ago, a discussion of interesting topics was held on [the 2007 summit list](<http://thunk.org/mailman/listinfo/ksummit-2007-discuss>). Many of the usual topics came around; there is always plenty of interesting development work going on in the kernel community. Andrew Morton [objected](<http://thunker.thunk.org/pipermail/ksummit-2007-discuss/2007-May/000306.html>) to many of the topics under discussion, though, saying that the summit was not the appropriate venue to talk about them: 

My overall take on kernel summit: we spend far too much time talking about technical stuff. There is little benefit in doing this: we conduct technical discussions over email and we do it well, and there are many very good reasons for doing it that way.... We fly halfway around the world to yap on about dentry cache scalability? Spare me, we'd get more done by staying home. 

Andrew's conclusion, which was seconded by a number of other developers, was that the process-oriented discussions are always more interesting and useful than the deep technical sessions. Discussions of virtualization, memory management, or device drivers will always be uninteresting to a significant part of the group, and they do not necessarily add much over what can be done with email. But the process-oriented talk affects everybody and is much harder to do electronically. 

So this year's agenda is more high-level than in previous years. That does not mean that there will be no technical talk, though. Some of the more technical sessions will cover: 

  * Reports from mini-summits. The kernel is a big program, and developers often find that subsystem-specific questions are better addressed in smaller groups. At the summit, attendees from some recent mini-summits (covering power management, filesystems, storage, and virtualization, at least) will report back to the larger group. 

  * Real time and scheduler issues are on the agenda because there are some big decisions to make. While much of the real-time tree has found its way into the mainline, some of the more disruptive chunks (sleeping spinlocks, threaded interrupt handlers) remain outside. Also outside of the mainline is the syslets/threadlets patch set. Hopefully some decisions will be made on whether these features should be merged, and, if so, what needs to be done to get them into shape. 

  * There are a number of memory management issues out there, including the variable page and variable block size patches, approaches to deadlock avoidance, scalability work, and more. Also on the agenda is the more process-oriented question of why memory management patches are so hard to get into the mainline. 

  * Virtualization has fallen off the agenda because most of the kernel-level work in this area has already been merged. The containers developers are just getting going, though, and there are a lot of questions about what their final destination is thought to be. A full containers implementation could impose significant overhead - on developers and on run-time performance - and could prove hard to sell. 

That's about it for the serious technical talks; everything else will have a higher-level focus. The summit will start with a panel of distributor kernel maintainers. To a great extent, distributors are the immediate customers for the kernels that the developers put out; those distributors are then charged with getting mainline releases into a condition that allows it to be shipped to users. Distributor kernel maintainers tend to be on the front line when things go wrong; they always hear about all the problems. This panel will be a chance for those maintainers to talk about the quality of the kernels they are getting from the mainline and how things could be made to work better. 

Once upon a time, the kernel stood alone and presented services to the system by way of the system call interface. In current systems, instead, users see a view of the system which is created by a whole set of utilities, including the C library, udev, HAL, and more. Interactions between these low-level components and the kernel is not always as smooth as it could be, and, despite the best efforts of the kernel development community, kernel releases have been known to occasionally break utilities like udev. The "greater kernel ecosystem" session will cover these issues and the general question of making the system as a whole work better together. Establishing better control over the user-space API is likely to come up, though the problem remains difficult. 

There is a half-hour session on developer relations. The kernel development community is visibly growing, and that is generally a good thing. Ensuring the continued health of kernel development requires bringing in a steady stream of new developers - from all over the world. This session will be the place to talk about how that can be done, and how participation from under-represented parts of the world can be improved. 

Andrew Morton gets an hour to pound the table on kernel quality and related issues. There still appears to be a consensus among the developers that the kernel is not getting buggier, but that view is not universally held. Everybody agrees that fewer bugs would be a good thing, though. So topics like bug tracking, fixing the reviewer shortage, possible stabilization releases, and so on, are likely to come up in this session. 

Documentation is, inevitably, on the agenda - everybody wants more of it, but, somehow, it fails to just show up on its own. Last year there was some talk of imposing documentation requirements on new patches, but few people took the idea all that seriously. So maybe some different ideas for improving the situation will come about this time around. Also on the list may be the area of managing translations - an area of increasing interest - and [standardizing kernel messaging](<http://lwn.net/Articles/238948/>). 

Various other process-oriented questions have been swept into a session late on the second day. Are big code cleanups worth it? How can we improve our handling of large patches which affect a number of different subsystems? How do we deal with problematic maintainers? And, in general, is the kernel process going too fast? But perhaps the discussion will be dominated by Andrew Morton's suggestion that the developers form a union and demand a massive pay raise. 

There are other sessions on the agenda as well; see the posted version for the full list. Whenever a group of this nature comes together, interesting things are bound to come out of it. Tune into LWN around September 6 for coverage from the event.
