---
title: The 2008 Linux Kernel Summit
url: https://lwn.net/Articles/298587/
date: "September 16, 2008"
category: Kernel Summit
author: "By Jonathan Corbet September 16, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
September 16, 2008

[![\[laptop surgery\]](https://static.lwn.net/images/conf/lpc-ks-2008/lt-laptop-sm.jpg)](<https://lwn.net/Articles/298588/>) The 2008 Linux Kernel Summit was held September 15 and 16 in Portland, Oregon, immediately prior to the Linux Plumbers Conference. At this invitation-only meeting, some 80 developers discussed a number of issues relevant to the kernel and its future development. The following reports were written by Jonathan Corbet, who attended the event and was a member of its program committee. 

This reporting was sponsored by LWN's subscribers; if you appreciate this kind of content, please consider [subscribing to LWN](</op/Subscriptions.lwn>) and helping us create more of it. 

### Day 1

The sessions held on the first day were: 

  * [Linux 3.0](<https://lwn.net/Articles/298510/>): should the developers do a Linux 3.0 release with a focus on dumping older, unneeded code? 

  * [Minisummit reports](<https://lwn.net/Articles/298521/>): reports from gatherings of power management, wireless networking, and containers developers. 

  * [When should drivers be merged?](<https://lwn.net/Articles/298570/>) A wide-ranging discussion on the trade-offs between getting drivers into the kernel quickly and waiting until they are up to kernel coding standards. 

  * [Filesystem and block layer interaction](<https://lwn.net/Articles/298589/>); what contemporary file systems need to be able to get the most out of storage devices. 

  * [Cross-subsystem issues](<https://lwn.net/Articles/298591/>); how do we evolve subsystems which are heavily used by several other parts of the kernel? 

  * [Tools](<https://lwn.net/Articles/298592/>), and the new Patchwork tool in particular. 

  * [Bootstrap code](<https://lwn.net/Articles/298593/>). Why does every distributor throw together its own initrd/initramfs code, and can that situation be improved? 

  * [Kernel quality and release process](<https://lwn.net/Articles/298596/>), various discussions on how to produce better kernels and a near-decision to move to a one-week merge window. 

> [![\[group photo\]](https://static.lwn.net/images/conf/lpc-ks-2008/ks-group-sm.jpg)](<https://lwn.net/Articles/298798/>)

### Day 2

  * [Tracing](<https://lwn.net/Articles/298685/>). A lengthy discussion on user requirements for kernel tracing and how those requirements might eventually be met. 

  * [Documentation](<https://lwn.net/Articles/298832/>). We always want more and better documentation, but what documentation would be most useful to the development community? 

  * There was a brief bug-fixing session aimed at the top entries on the [KernelOops.org](<http://kerneloops.org/>). Over the course of half an hour, the developers were able to fix 13 of the top 14 bugs. It was widely agreed that this was a productive use of time which will probably be repeated at future events. 

  * [More minisummit reports](<https://lwn.net/Articles/298835/>) covering virtualization, networking, and kernel bloat. 

  * [All about threads](<https://lwn.net/Articles/298840/>); kernel thread pools and threaded interrupt handlers in particular. 

  * [Projects with large user-space components](<https://lwn.net/Articles/298842/>); how can we make it easier for the direct rendering infrastructure project to work with the mainline kernel? 

  * Rafael Wysocki led a section on the new suspend/resume infrastructure. Most of that talk was concerned with the API, which was [covered here](<http://lwn.net/Articles/274008/>) back in March, so it will not be written up again now. Some changes will likely be made; stay tuned to LWN for the details. 

Linus did ask the crowd how many people were still unable to suspend their laptops. The number of hands raised was quite small; things have clearly gotten better in this area. 

  * [Fixing the Kernel Janitors Project](<https://lwn.net/Articles/298854/>). How can we do a better job of bringing new developers into the kernel community? 

The closing party (which was also the Linux Plumbers Conference opening party) was the venue chosen for the annual election of members to the Linux Foundation's Technical Advisory Board. The move out of the regular kernel summit sessions was intended to allow a wider group of people to participate in the election. It would appear to have been successful in that regard; there were record numbers of both candidates and voters. The board members elected this time around were James Bottomley, Kristen Carlson Accardi, Chris Mason, Dave Jones, Chris Wright, and Christoph Hellwig. Christoph was elected to a one-year term; all of the others will serve two-year terms. 

Next year's kernel summit is currently scheduled for October 18 to 20 in Tokyo, Japan.
