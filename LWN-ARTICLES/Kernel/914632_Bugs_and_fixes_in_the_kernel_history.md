---
title: Bugs and fixes in the kernel history
url: https://lwn.net/Articles/914632/
date: "December 8, 2022"
category: "Development model-Kernel quality"
author: "By Jonathan Corbet December 8, 2022"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 8, 2022

Each new kernel release fixes a lot of bugs, but each release also introduces new bugs of its own. That leads to a fundamental question: is the kernel community fixing bugs more quickly than it is adding them? The answer is less than obvious but, if it could be found, it would give an important indication of the long-term future of the kernel code base. While digging into the kernel's revision history cannot give a definitive answer to that question, it can provide some hints as to what that answer might be. 

#### Tagged fixes

In current kernel practice, a developer who fixes a bug is expected to include a Fixes tag in the patch description identifying the commit that introduced that bug. This is a relatively recent practice; while various forms of Fixes tags had appeared in commits for some time, the first patch using the current form with the hash of the offending commit appears to be [this revert](<https://git.kernel.org/linus/c511851de162>) from Rafael Wysocki for 3.12 in October 2013. In that release, only two commits identified buggy commits from previous releases, but the use of this tag grew quickly in subsequent development cycles. The 6.0 kernel release included 2,784 commits with Fixes tags, 2,112 of which identified commits from previous releases as the source of the bug being fixed (the remaining commits fixed bugs that had been introduced in 6.0 and thus never appeared in a released kernel). 

Thus, in theory, one can simply count Fixes tags over a development cycle to see how many bugs from previous releases were fixed. Then, looking at subsequent releases, the Fixes tags can be used to see how many bugs were introduced in that cycle and fixed in later cycles. If the number of bugs fixed in a development cycle regularly exceeds the number of bugs introduced in that cycle, then chances are good that the kernel is getting better over time. The idea is simple, but runs into some practical difficulties that will be explored later on. We can start with a plot showing how the above analysis comes out: 

![\[bugs introduced and fixed per release\]](https://static.lwn.net/images/2022/release-fixes.svg)

(This data can also be viewed in tabular form on [this page](<https://lwn.net/Articles/914479/>)). 

In the above plot, the thicker lines are counts of Fixes tags; so the brown "bugs introduced" line is the number of times that a commit in a given release was identified by a Fixes tag in subsequent releases, while the green "bugs fixed" line shows the number of Fixes tags in a given release identifying buggy commits in previous releases. The thinner lines are instead counting commits: "buggy commits introduced" is the number of commits in a given release that were later fixed, and "commits fixed" is the number of commits from previous releases that were fixed in a given release. 

The two sets of numbers differ for a simple reason: some commits are sufficiently buggy that they need to be fixed more than once — a topic we'll return to shortly. There is an interesting difference here, though: in any given development cycle, the number of bugs fixed tracks closely with the number of commits fixed, but there is a big difference between the number of bugs introduced and the number of buggy _commits_ introduced. What we can conclude from this difference is that commits that introduce a lot of bugs require multiple development cycles for all of those bugs to be fixed. It is rare to see a lot of fixes to the same commit in any one development cycle. 

Can this plot answer the question posed at the beginning of this article, though? A naive reading shows that the lines cross and that, thus, number of bugs fixed exceeds the number of bugs introduced as of the 5.1 release. But that result must clearly be taken with a fair amount of salt. As has been seen in other recent [examinations of Fixes tags](<https://lwn.net/Articles/909625/>), bugs lurk in the kernel for a long time. Kernel developers are still finding and fixing bugs introduced early in the 2.6 era — and before. So the "bugs introduced" numbers for recent kernels are clearly too low, as can be seen by the fact that those lines head toward zero for the most recent releases. 

The number of bugs introduced does appear, though, to level out in the range of 1,200 to 1,400 per release; this can be seen in the older releases, where the numbers are unlikely to change much at this point. That trend seems to continue through about 5.8 or so, after which the curve drops down and clearly does not reflect long-term reality. Should this pattern hold — something only time will tell — then the point where the curves cross may move, but it seems likely to remain in the early 5.x era. If that is truly the case then, in recent times at least, the kernel community may well be fixing more bugs than it is introducing. 

What might have caused the situation to change? Your editor does not know but can wave his hands as well as anybody else. One possibility is improved development tools and, especially, the increased use of fuzz testing to turn up old bugs and prevent new ones. The slow but steady growth in the kernel's (still inadequate) testing infrastructure will have helped. Increased insistence on patch review may have helped to keep the number of bugs introduced roughly constant even as the volume of code going into the kernel has increased. Or perhaps none of the above applies. 

It is also almost certainly true that developers have become more disciplined about adding Fixes tags, causing more bug fixes to actually be counted as such while not actually reflecting a change in the rate at which fixes are happening. In general, Fixes tags may be the best proxy we have for actual bug counts, but they are still an inaccurate metric; it depends on developers to carefully add them and to correctly identify the commits that introduce bugs. 

#### The buggiest commits

One thing those tags might do reliably, though, is to identify the buggiest commits in the kernel's history. Remember that some commits require more than one fix over time; some of them require quite a few more than one. Here is a table of the most-fixed commits during the Git era: 

> Commit| Fixes| Description  
> ---|---|---  
> [1da177e4c3f4](<https://git.kernel.org/linus/1da177e4c3f4>) | [355](<https://lwn.net/Articles/914655/#1da177e4c3f4>) | Linux-2.6.12-rc2  
> [e126ba97dba9](<https://git.kernel.org/linus/e126ba97dba9>) | [70](<https://lwn.net/Articles/914655/#e126ba97dba9>) | mlx5: Add driver for Mellanox Connect-IB adapters  
> [8700e3e7c485](<https://git.kernel.org/linus/8700e3e7c485>) | [65](<https://lwn.net/Articles/914655/#8700e3e7c485>) | Soft RoCE driver  
> [46a3df9f9718](<https://git.kernel.org/linus/46a3df9f9718>) | [54](<https://lwn.net/Articles/914655/#46a3df9f9718>) | net: hns3: Add HNS3 Acceleration Engine & Compatibility Layer Support  
> [9d71dd0c7009](<https://git.kernel.org/linus/9d71dd0c7009>) | [42](<https://lwn.net/Articles/914655/#9d71dd0c7009>) | can: add support of SAE J1939 protocol  
> [76ad4f0ee747](<https://git.kernel.org/linus/76ad4f0ee747>) | [38](<https://lwn.net/Articles/914655/#76ad4f0ee747>) | net: hns3: Add support of HNS3 Ethernet Driver for hip08 SoC  
> [604326b41a6f](<https://git.kernel.org/linus/604326b41a6f>) | [38](<https://lwn.net/Articles/914655/#604326b41a6f>) | bpf, sockmap: convert to generic sk_msg interface  
> [1738cd3ed342](<https://git.kernel.org/linus/1738cd3ed342>) | [38](<https://lwn.net/Articles/914655/#1738cd3ed342>) | net: ena: Add a driver for Amazon Elastic Network Adapters (ENA)  
> [e1eaea46bb40](<https://git.kernel.org/linus/e1eaea46bb40>) | [35](<https://lwn.net/Articles/914655/#e1eaea46bb40>) | tty: n_gsm line discipline  
> [e7096c131e51](<https://git.kernel.org/linus/e7096c131e51>) | [34](<https://lwn.net/Articles/914655/#e7096c131e51>) | net: WireGuard secure network tunnel  
> [1c1008c793fa](<https://git.kernel.org/linus/1c1008c793fa>) | [33](<https://lwn.net/Articles/914655/#1c1008c793fa>) | net: bcmgenet: add main driver file  
> [d5c65159f289](<https://git.kernel.org/linus/d5c65159f289>) | [29](<https://lwn.net/Articles/914655/#d5c65159f289>) | ath11k: driver for Qualcomm IEEE 802.11ax devices  
> [c0c050c58d84](<https://git.kernel.org/linus/c0c050c58d84>) | [27](<https://lwn.net/Articles/914655/#c0c050c58d84>) | bnxt_en: New Broadcom ethernet driver.  
> [c09440f7dcb3](<https://git.kernel.org/linus/c09440f7dcb3>) | [27](<https://lwn.net/Articles/914655/#c09440f7dcb3>) | macsec: introduce IEEE 802.1AE driver  
> [7724105686e7](<https://git.kernel.org/linus/7724105686e7>) | [26](<https://lwn.net/Articles/914655/#7724105686e7>) | IB/hfi1: add driver files  
> [d2ead1f360e8](<https://git.kernel.org/linus/d2ead1f360e8>) | [25](<https://lwn.net/Articles/914655/#d2ead1f360e8>) | net/mlx5e: Add kTLS TX HW offload support  
> [7733f6c32e36](<https://git.kernel.org/linus/7733f6c32e36>) | [25](<https://lwn.net/Articles/914655/#7733f6c32e36>) | usb: cdns3: Add Cadence USB3 DRD Driver  
> [726b85487067](<https://git.kernel.org/linus/726b85487067>) | [24](<https://lwn.net/Articles/914655/#726b85487067>) | qla2xxx: Add framework for async fabric discovery  
> [1e51764a3c2a](<https://git.kernel.org/linus/1e51764a3c2a>) | [24](<https://lwn.net/Articles/914655/#1e51764a3c2a>) | UBIFS: add new flash file system  
> [a49d25364dfb](<https://git.kernel.org/linus/a49d25364dfb>) | [24](<https://lwn.net/Articles/914655/#a49d25364dfb>) | staging/atomisp: Add support for the Intel IPU v2  
> [96c8395e2166](<https://git.kernel.org/linus/96c8395e2166>) | [24](<https://lwn.net/Articles/914655/#96c8395e2166>) | spi: Revert modalias changes  
> [3c4d7559159b](<https://git.kernel.org/linus/3c4d7559159b>) | [23](<https://lwn.net/Articles/914655/#3c4d7559159b>) | tls: kernel TLS support  
> [d7157ff49a5b](<https://git.kernel.org/linus/d7157ff49a5b>) | [23](<https://lwn.net/Articles/914655/#d7157ff49a5b>) | mtd: rawnand: Use the ECC framework user input parsing bits  
> [6a98d71daea1](<https://git.kernel.org/linus/6a98d71daea1>) | [22](<https://lwn.net/Articles/914655/#6a98d71daea1>) | RDMA/rtrs: client: main functionality  
> [3f518509dedc](<https://git.kernel.org/linus/3f518509dedc>) | [22](<https://lwn.net/Articles/914655/#3f518509dedc>) | ethernet: Add new driver for Marvell Armada 375 network unit  
> [ca6fb0651883](<https://git.kernel.org/linus/ca6fb0651883>) | [21](<https://lwn.net/Articles/914655/#ca6fb0651883>) | tcp: attach SYNACK messages to request sockets instead of listener  
> [ad67b74d2469](<https://git.kernel.org/linus/ad67b74d2469>) | [21](<https://lwn.net/Articles/914655/#ad67b74d2469>) | printk: hash addresses printed with %p  
> [c29f74e0df7a](<https://git.kernel.org/linus/c29f74e0df7a>) | [20](<https://lwn.net/Articles/914655/#c29f74e0df7a>) | netfilter: nf_flow_table: hardware offload support  
> [d2ddc776a458](<https://git.kernel.org/linus/d2ddc776a458>) | [20](<https://lwn.net/Articles/914655/#d2ddc776a458>) | afs: Overhaul volume and server record caching and fileserver rotation  
> [1a86b377aa21](<https://git.kernel.org/linus/1a86b377aa21>) | [20](<https://lwn.net/Articles/914655/#1a86b377aa21>) | vdpa/mlx5: Add VDPA driver for supported mlx5 devices  
  
One might wonder about what went wrong with Linux-2.6.12-rc2, which has been fixed (at last count) 355 times. That is, of course, the initial commit that started the Git era, so fixes identifying that commit are for bugs that were introduced prior to April 2005. Even in 2022, bugs of that vintage are still being found and fixed. 

After that, the conclusion to be drawn is not that surprising: the commits that need a lot of fixes tend to be the large ones that add a significant new subsystem. A lot of new code will inevitably bring a fair number of new bugs with it, and those bugs will need to be discovered and fixed over time. One interesting exception might be [ca6fb0651883](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ca6fb0651883>) ("tcp: attach SYNACK messages to request sockets instead of listener") which inserted 47 lines in 2015 and has been fixed 21 times since, most recently in February for 5.17. Also noteworthy is [96c8395e2166](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=96c8395e2166>) ("spi: Revert modalias changes"), which deleted six lines of code and has required 24 fixes thereafter. Beyond those, though, the commits needing a large number of fixes have been large in their own right. 

Perhaps more interesting is the fact that, of the 30 most-fixed commits shown above, 22 are related to networking (including InfiniBand). The networking subsystem is a large part of the kernel, but it is still a small piece of the whole and not the only subsystem that merges large patches. It's not clear why networking-related patches, in particular, would be more likely to need many fixes. 

Bugs are a fact of life in software development, unfortunately, and we are unlikely to be free of them anytime soon. If an optimistic reading of the data above reflects reality, though, then it is possible that the kernel-development community may have reached a point where it is fixing more bugs than it introduces. LWN will surely revisit this topic in the future to see how the situation evolves.
