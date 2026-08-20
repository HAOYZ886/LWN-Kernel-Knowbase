---
title: Merged (and to be merged) for 2.6.22
url: https://lwn.net/Articles/232247/
date: "May 2, 2007"
category: "Releases-2.6.22"
author: 
---

The 2.6.22 merge window has opened, with almost 2,000 changesets merged as of this writing. The merge process appears to have slowed somewhat; it may be that the level of traffic on linux-kernel is so high (even by linux-kernel standards) that nobody has time to deal with actual patches. Be that as it may, user-visible changes merged so far include: 

  * Lots of networking changes, including improvements to the forward receive timeout recovery ([RFC4138](<http://www.ietf.org/rfc/rfc4138.txt>)) implementation, a [YeAH-TCP congestion control [PDF]](<
     http://wil.cs.caltech.edu/pfldnet2007/paper/YeAH_TCP.pdf>) implementation, a [TCP Illinois congestion control](<http://www.ews.uiuc.edu/~shaoliu/tcpillinois/index.html>) implementation, and a new RxRPC secure socket layer (along with support for using RxRPC in the AFS filesystem). Also, the old, IPv4-only connection tracking code has been removed as per the feature removal schedule. 

  * The cfg80211 patches - a new, netlink-based interface for configuring wireless interfaces - have been merged. At the same time, the netlink version of the "wireless extensions" interface has been removed. 

  * The OCFS2 filesystem now has sparse file support. 

  * The [UBI](<http://www.linux-mtd.infradead.org/doc/ubi.html>) patch, which performs flash-aware partitioning and volume management, has been merged. 

  * New drivers for USB webcams based on zr364xx chipsets, AT26Fxxx dataflash devices, CM-X270-based NAND flash memory, Freescale SOC USB controllers, and Marvell Libertas 802.11 adaptors (used in the OLPC system). 

It's also worth noting that the [IVTV video driver](<http://ivtvdriver.org/index.php/Main_Page>), long out of the mainline, has finally been merged. ""It took three core maintainers, over four years of work, eight new i2c modules, eleven new V4L2 ioctls, three new DVB video ioctls, a Sliced VBI API, a new MPEG encoder API, an enhanced DVB video MPEG decoding API, major YUV/OSD contributions from Ian and John, web/wiki/svn/trac support from Axel Thimm, (hardware) support from Hauppauge, support and assistance from the v4l-dvb people and the many, many users of ivtv to finally make it possible to merge this driver into the kernel."" 

  * A new "sony-laptop" layer which replaces sonypi and provides better Sony support. The old "ibm_acpi" module has been renamed "thinkpad-acpi," and it features improved support for those laptops. 

  * The CFQ I/O scheduler has been reworked. Taking inspiration from the CFS CPU scheduler, it now uses a red-black tree to sort pending requests by expected execution time and track them. 

Changes visible to kernel developers include: 

  * The `eth_type_trans()` function now sets the `skb->dev` field, consistent with how similar functions for other link types operate. As a result, many Ethernet drivers have been changed to remove the (now) redundant assignment. 

  * The header fields in the `sk_buff` structure have been renamed and are no longer unions. Networking code and drivers can now just use `skb->transport_header`, `skb->network_header`, and `skb->skb_mac_header`. There are new functions for finding specific headers within packets: `tcp_hdr()`, `udp_hdr()`, `ipip_hdr()`, and `ipipv6_hdr()`. 

  * Also in the networking area: the packet scheduler has been reworked to use `ktime` values rather than `jiffies`. 

Those who are curious about what else might get in to 2.6.22 can have a look at Andrew Morton's [2.6.22 merge plans](<http://lwn.net/Articles/232432/>) document. Interestingly, [Lguest](<http://lwn.net/Articles/218766/>), the [signalfd](<http://lwn.net/Articles/225714/>) work, and the [SLUB allocator](<http://lwn.net/Articles/229984/>) are all planned for merging, but all have become less certain since: 

  * There have been some complaints that Lguest has not been sufficiently reviewed. Since this development is independent and will not bother those who do not use it, the concerns are less likely to delay its inclusion. 

  * Signalfd has a new competitor in the form of the [pollfs](<http://lwn.net/Articles/232685/>) patch. Pollfs takes takes a different approach to many of the same problems and throws in polling for futex operations as well. It is far from clear that pollfs is better (some of the early reviews have been on the unfavorable side), but the process of figuring out whether that is true could delay signalfd past the closing of the merge window. 

  * The SLUB allocator has also been subject to [concerns](<https://lwn.net/Articles/232774/>) that it has not been sufficiently tested for such a fundamental change. Additionally, there seems to be a difference of goals between Andrew Morton (who would like to see SLUB eventually replace the current slab allocator) and SLUB developer Christoph Lameter, who had seen the two coexisting indefinitely. Chances are these issues will get worked out and SLUB will go in as scheduled. 

There are a few things of interest which are not on Andrew's list. The reiser4 filesystem seems certain to sit out (at least) another cycle, despite a resurgence in interest in getting it ready for inclusion. Xen is not mentioned, but it seems that, behind the scenes, it is being worked on. So Xen could actually show up before the merge window closes. There will be no major scheduler rework in 2.6.22; it's too soon for any of those patches to go in. The anti-fragmentation patches look likely to wait a little longer; Andrew worries that they still haven't seen enough review and benchmarking despite many iterations over a few years. The [integrity management patches](<http://lwn.net/Articles/227937/>) are considered to be unready and will not be merged. 

Beyond that, there will be doubtless be surprises over the next week or so; stay tuned.
