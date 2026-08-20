---
title: The end of the 4.6 merge window
url: https://lwn.net/Articles/681539/
date: "March 30, 2016"
category: "Releases-4.6"
author: "By Jonathan Corbet March 30, 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 30, 2016

Linus released [4.6-rc1](<https://lwn.net/Articles/681470/>) and closed the merge window on March 26, one day earlier than might have been expected; at that time, he cited the size of this development cycle as one of the reasons for the early release. This merge window ended with 12,172 non-merge changesets being pulled into the mainline repository, which is indeed a significant number — indeed, it is the busiest merge window in the history of the kernel project. The only other cycles that came close are 4.2 (12,092 changesets during the merge window) and 3.15 (12,034). Given that there are some significant cleanup patches that may go in shortly, 4.6 has a good chance of being the busiest development cycle ever. 

Just over 1,000 of those changesets were merged after [last week's summary](<https://lwn.net/Articles/680566/>); as was suggested at the time, the pace slowed toward the end. Still, some significant changes were merged, including: 

  * The infrastructure for [tracing histogram triggers](<https://lwn.net/Articles/635522/>) has been merged, though the complete feature is still waiting for a bit more testing. 

  * The pNFS subsystem (and the NFS server in particular) now supports a "SCSI layout" mode. See [`pnfs-scsi-server.txt`](<https://lwn.net/Articles/681541/>) for some more information. 

  * The [out-of-memory reaper](<https://lwn.net/Articles/668126/#reaper>) patches have been merged. This patch set makes the kernel more aggressively take memory away from processes being terminated by the out-of-memory killer, hopefully making the out-of-memory response faster and more reliable. 

  * After a long time and many rounds of review, the [OrangeFS](<https://lwn.net/Articles/643165/>) distributed filesystem has been merged. There are seemingly a few issues to be worked on still, but the code was deemed to be in good enough shape to go into the mainline. 

  * New hardware support includes: Qualcomm IPQ4019 global clock controllers, Qualcomm NAND flash controllers, Texas Instruments dm814x ADPLL clocks, and Texas Instruments Message Manager hardware mailboxes. 

Additionally, last week's summary missed the addition of a number of system-on-chip platforms and boards, including Allwinner A83T, Annapurna Labs Alpine v2, Axis Artpec-6, Broadcom Vulcan, NXP i.MX 6QuadPlus, ARM Juno R2 development boards, Marvell Armada 3700, 7XXX and 8XXX, MediaTek MT7623, Amlogic S905 (Meson GXBaby), Qualcomm Snapdragon MSM8996 TI KeyStone K2G, and SocioNext UniPhier PH1. 

  * The slab memory allocator now has support for the [KASAN](<https://lwn.net/Articles/612153/>) debugging tool. 

If the usual (for recent years) 63-day pattern holds, the final 4.6 release can be expected on May 15, though unforeseen problems or opportunities to do some good diving can always cause delays.
