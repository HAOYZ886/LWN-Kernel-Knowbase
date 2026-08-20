---
title: The 2012 Kernel Summit
url: https://lwn.net/Articles/513789/
date: "August 29, 2012"
category: Kernel Summit
author: "By Michael Kerrisk and Jake Edge August 29, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Michael Kerrisk and Jake Edge**  
August 29, 2012

The 2012 Kernel Summit was held in San Diego, CA, USA, over three days, 27-29 August. As with the [2011 Kernel Summit](<https://lwn.net/Articles/KernelSummit2011/>) in Prague (and following on from [discussions at the 2010 Kernel Summit](<https://lwn.net/Articles/413095/>)), the 2012 summit followed a different [format](<https://lwn.net/Articles/502181/>) from the ten previous summits. For 2012, the event took the form of an invitation-only plenary-session day followed by two days of minisummits and additional technical sessions shared with the co-located 2012 Linux Plumbers Conference that kicked off on 29 August; the agenda for days 1 and 3 can be found [here](<https://sites.google.com/site/kernelsummit2012/agenda>). (The [ARM minisummit](<https://lwn.net/Articles/513756/>) was something of an exception to this format: it ran for two days, starting on the same day as the plenary sessions.) 

> [![\[2012 Kernel Summit group photo\]](https://static.lwn.net/images/conf/2012/ks/ks2012-group-sm.jpg)](<https://lwn.net/Articles/513945/>)

#### Main summit, day 1

The first day of the Kernel Summit, on 27 August, consisted of plenary sessions attended by around 80 invitees. Among the topics were the following: 

  * [The future of kernel regression tracking](<https://lwn.net/Articles/513801/>); the kernel development community is in strong agreement on the value of regression tracking, and is currently looking for some person(s) to take up this high-profile work. 

  * [Supporting old/oddball architectures, tool chains, and devices](<https://lwn.net/Articles/514004/>): how long must we support ancient hardware and software, and how do we leave it behind? 

  * [Regression testing](<https://lwn.net/Articles/514054/>); how can we do a better job of finding bugs before they bite users? 

  * [Distributions and upstream](<https://lwn.net/Articles/514754/>); what can kernel developers do to make life easier for their main customers — the distributors? 

  * [Lightning talks](<https://lwn.net/Articles/514778/>): quick sessions on RCU callbacks and Smatch. 

  * [Kernel build and boot testing](<https://lwn.net/Articles/514278/>); a new framework for quickly finding regressions. 

  * [Android upstreaming](<https://lwn.net/Articles/514901/>): the ongoing process of getting the Android kernel code into the mainline. 

  * [Improving the maintainer model](<https://lwn.net/Articles/514893/>); do our subsystem maintainers scale? 

  * [Stable kernel management](<https://lwn.net/Articles/515528/>); how is the stable process working? 

  * [Tracing and debugging](<https://lwn.net/Articles/514898/>), and how to get better oops output in particular. 

  * [Linux-next](<https://lwn.net/Articles/515529/>) and related improvements to the development process. 

#### Main summit, day 2

  * [The memcg/mm minisummit](<https://lwn.net/Articles/516439/>) covering a wide range of topics related to memory management. 

#### Main summit, day 3

  * [Module signing](<https://lwn.net/Articles/515007/>); toward a way to finally get this feature into the kernel. 

  * [Kernel summit feedback](<https://lwn.net/Articles/515209/>); how did the event work out this year, and what changes should be made for future years? 

#### ARM minisummit, day 1

The first day of this year's Kernel Summit coincided with day one of the ARM minisummit. Given that the "minisummit" spanned two days, there was talk of false advertising, but there was lots to cover. 

  * [Secure monitor API](<https://lwn.net/Articles/513756/>): how best to support the secure monitor mode across a wide variety of processors. 

  * [Stale platform deprecation](<https://lwn.net/Articles/513941/>): some ARM platform support has clearly not been used for years; how do we clean out the cruft? 

  * [Virtualization](<https://lwn.net/Articles/513940/>) is coming to ARM, but brings some issues of its own. 

  * [DMA mapping](<https://lwn.net/Articles/513939/>) has seen a lot of work in the last year, but there is still a fair amount to be done. 

#### ARM minisummit, day 2

  * [Process review for the arm-soc tree](<https://lwn.net/Articles/513938/>): how well is this tree working toward the goal of cleaning up the ARM architecture code? 

  * [Toward a single kernel image](<https://lwn.net/Articles/513952/>): what needs to be done to get a single kernel that boots on multiple ARM processor families? 

  * [AArch64](<https://lwn.net/Articles/514062/>): the current status of 64-bit support for the ARM architecture. 

  * [A big.LITTLE update](<https://lwn.net/Articles/514063/>); how can the kernel support this novel architecture? 

  * [DMA issues](<https://lwn.net/Articles/514064/>) and how to best support generic DMA engines in particular. 

#### Linux Security Summit

  * [Secure Boot](<https://lwn.net/Articles/515596/>): keynote from Matthew Garrett. 

  * [Secure Linux containers](<https://lwn.net/Articles/515034/>): using SELinux to create sandboxed containers. 

  * [Integrity for directories and special files](<https://lwn.net/Articles/516334/>): extending the Integrity Measurement Architecture (IMA) to handle directories and other special files. 

  * [DNSSEC](<https://lwn.net/Articles/516797/>): a look at the "cryptographically secured globally distributed database" for domain names and more. 

  * [Security modules and RPM](<https://lwn.net/Articles/518265/>): expanding the hooks in RPM to support Smack and other security technologies. 

  * [Kernel security subsystem reports](<https://lwn.net/Articles/517384/>): reports from subsystem maintainers. 

#### Notes from others

  * [PCI minisummit](<https://lwn.net/Articles/514113/>), notes posted by Bjorn Helgaas. 

  * [ARM minisummit](<https://lwn.net/Articles/514159/>), posted by Will Deacon. 

  * [Media workshop notes, part 1](<https://lwn.net/Articles/514527/>) by Mauro Carvalho Chehab. 

  * [The realtime microconference](<https://lwn.net/Articles/515828/>) from LPC, courtesy of Darren Hart. 

#### Acknowledgments

Michael would like to thank the Linux Foundation for supporting his travel to San Diego for this event; Jake would like to thank LWN subscribers for the same.
