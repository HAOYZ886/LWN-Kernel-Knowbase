---
title: "KS2010: Minisummit reports"
url: https://lwn.net/Articles/413036/
date: "November 3, 2010"
category: "Networking-Networking summits; Nftables"
author: "By Jonathan Corbet November 3, 2010 2010 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 3, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

The kernel is made up of many subsystems which qualify as major programs on their own. These subsystems tend to have technical issues which are best discussed in smaller groups of concerned developers; thus, there tend to be several focused "minisummits" over the course of the year. The Kernel Summit traditionally provides a slot for reports from these minisummits, and 2010 was no exception. 

Dave Miller reported on the **netfilter** workshop, recently held in Seville. Many topics were covered there, including IP virtual server (IPVS) support. Work is progressing in this area, with better support for namespaces, DNAT support, and a new persistence engine for the SIP protocol. Also in the works is connection synchronization support, enabling the operation of hot backup servers. 

Ingress bandwidth shaping is another area of development; it's not something we support well now. The existing options require out-of-tree patches. Also out of tree is the "xtables add-ons and blobs" repository maintained by Jan Engelhardt, which contains stuff which is "obscure but useful to somebody." Even more out of tree is the "OpenOffice.org firewall," a means for importing iptables statistics into OpenOffice.org which was implemented as a joke in response to silly statements by a French politician. 

There was a mention of ipset, which optimizes netfilter rules which match sets of addresses. Ipset lacks IPv6 support and needs a user interface rewrite; much of this work has now been completed. Ipset also uses the new libmnl netlink library, the third such library out there. Libmnl was created in response to deficiencies seen in libnetlink (which is too tightly tied to iproute2) and libnl (which is too abstracted and hides useful details). 

[Nftables](<https://lwn.net/Articles/324989/>), a netfilter replacement based on an in-kernel virtual machine, was covered here in 2009; since then, this project has gone quiet. It came up at the workshop, though; there is interest in it because it would allow the eventual elimination of much of the existing code and could also be used in other parts of the stack where packet classification is done (the packet scheduler, for example). It could become, in other words, a general packet classification mechanism for the networking layer. There is nervousness about adding interpreters to the kernel which nftables will have to overcome, but that can probably be done. 

Other topics mentioned included tproxy, which gained IPv6 support in 2.6.37, and [Open vSwitch](<http://openvswitch.org/>), a "dumb switch" with most of the smarts in user space. 

James Bottomley discussed the **Storage, filesystems and memory management** minisummit held in August. That event was [covered](<https://lwn.net/Articles/399148/>) [here](<https://lwn.net/Articles/399313/>) in detail at the time; readers are referred there for more detail. 

Len Brown talked about the **power management** meeting, also held in August. Len's notes on the event were [published here](<https://lwn.net/Articles/400465/>); that will be the place to look for a complete report. 

There have been two **embedded** minisummits held in the last couple of months, one in Tokyo and the other in Cambridge. There was a lot of talk at these events about "version spread," the wide variety of kernels used by various embedded vendors. A commitment was made, apparently, to get more out-of-tree stuff merged upstream. Some vendors are said to be actively interested in working closer with upstream; Google's merging of the Tegra architecture code was given as an example. 

There was apparently much complaining about the lack of support from system-on-chip vendors. 

Several vendors have agreed to standardize on an "embedded kernel flag version" for long-term maintenance. The enterprise vendors standardized on 2.6.32 with good effect; the embedded side wants to do the same with 2.6.35. Someday it might be nice if everybody picked the same kernel, but that's a task for the future. There is also a renewed push to get the yaffs2 filesystem cleaned up and merged; the recent patch posting was a bit of a surprise to the people working on this task, though. 

A **wireless networking** summit was held in San Francisco in September; John Linville gave the report. The big issue in wireless appears to be [WiFi Direct](<http://en.wikipedia.org/wiki/Wi-Fi_Direct>), a set of protocols for _ad hoc_ networking and resource discovery. The feature is likely to be called "WiFi peer-to-peer" in Linux to avoid the use of the trademarked name. It's a set of Bluetooth-like protocols which allow things like printer sharing; someday it may eliminate the need for access points altogether. Much of this work has been done, but the user-space interface work is not yet there. 

Mauro Carvalho Chehab reported on the **Video4Linux** minisummit. There was, it seems, a lot of attendance from system-on-chip companies. That reflects a general change in emphasis in that subsystem; it was initially focused on desktop video, but embedded is taking over. 

The Video4Linux1 API was implemented in 1999; it has been slated for removal ever since Video4Linux2 hit the scene in 2002. In 2011, it's actually happening: the V4L1 API will be removed in 2.6.38. There will be user-space support for this API by way of an add-on library; the community will also take on support for [camorama](<http://camorama.fixedgear.org/>), one of the few remaining V4L1 applications. 

There was talk of the new "media controller" API, meant to facilitate the configuration of streams within SoC systems. There are no users of this API yet, so Mauro is holding it out of the kernel for now; it may be merged in 2.6.38. The related "mediabus" API has gone in for 2.6.37. Also merged over the last year is the large set of out-of-tree infrared controller drivers; these are being migrated toward the new remote controller subsystem. A future issue is supporting the remote controller protocol added to HDMI 1.3. Another new API is videobuf2, which is currently being reviewed and should be merged soon. 

Finally, Mauro talked about the problem of routing video output to framebuffer devices. There is no consensus on how this should be done, and the existing overlay code is unmaintained. There is interest in this capability, though, so the problem is not going to go away. 

There was also a brief update on the **tracing** minisummits presented by Masami Hiramatsu. New work in tracing includes the [kernelshark](<http://rostedt.homelinux.com/kernelshark/>) tool and improvements to the `perf probe` command. Areas of ongoing interest include the merging of uprobes and utrace, and user-space tracing in general. Linus commented that one of the best features of `perf` is the ease with which it can be used; there is nothing similar for tracing. His hope is that tracing will move in that direction over time. 

[Next: Security](<https://lwn.net/Articles/413102>).
