---
title: Development statistics for 2.6.31
url: https://lwn.net/Articles/348445/
date: "August 26, 2009"
category: "Releases-2.6.31"
author: "By Jonathan Corbet August 26, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 26, 2009

The Linux Foundation recently [announced](<http://www.marketwire.com/press-release/The-Linux-Foundation-1032809.html>) the release of an updated version of its kernel authorship report, co-written by LWN editor Jonathan Corbet. The information there is interesting, but, since it stops with the 2.6.30 kernel, it also is ancient history at this point. 2.6.30 came out two full _months_ ago, after all. LWN readers, certainly, are used to more current information. Since 2.6.31 is getting close to ready, it seems like the right time to look at this development cycle and see where the code came from. 

As of this writing (just after the release of [2.6.31-rc7](<http://lwn.net/Articles/348233/>)), the 2.6.31 development cycle had seen the incorporation of 10,663 non-merge changesets from 1,146 individual developers. These patches added almost 903,000 lines of code and removed just over 494,000 lines, for a net growth of just over 408,000 lines. According to Rafael Wysocki's [August 25 report](<http://lwn.net/Articles/348818/>), this work introduced 108 regressions into the kernel, 26 of which still lack a resolution. 

The largest individual contributors in the 2.6.31 development cycle were: 

> Most active 2.6.31 developers  
> ---  
> | By changesets  
> ---  
> Ingo Molnar| 276| 2.6%  
> Peter Zijlstra| 260| 2.4%  
> Paul Mundt| 204| 1.9%  
> Takashi Iwai| 150| 1.4%  
> Bartlomiej Zolnierkiewicz| 149| 1.4%  
> Steven Rostedt| 139| 1.3%  
> Tejun Heo| 134| 1.3%  
> Johannes Berg| 133| 1.2%  
> Magnus Damm| 119| 1.1%  
> Mike Frysinger| 115| 1.1%  
> roel kluin| 105| 1.0%  
> Greg Kroah-Hartman| 101| 0.9%  
> Erik Andren| 100| 0.9%  
> Paul Mackerras| 85| 0.8%  
> Mark Brown| 85| 0.8%  
> Bill Pemberton| 82| 0.8%  
> Jaswinder Singh Rajput| 79| 0.7%  
> Ben Dooks| 72| 0.7%  
> Joe Perches| 72| 0.7%  
> Alexander Beregalov| 71| 0.7%  
> | By changed lines  
> ---  
> Bartlomiej Zolnierkiewicz| 220749| 18.3%  
> Jerry Chuang| 78441| 6.5%  
> Forest Bond| 50834| 4.2%  
> David Daney| 40052| 3.3%  
> Jerome Glisse| 38604| 3.2%  
> Vlad Zolotarov| 23260| 1.9%  
> Ingo Molnar| 22614| 1.9%  
> James Smart| 19209| 1.6%  
> Bill Pemberton| 17249| 1.4%  
> dmitry pervushin| 14532| 1.2%  
> Greg Kroah-Hartman| 13234| 1.1%  
> Wai Yew CHAY| 12741| 1.1%  
> Michael Chan| 11887| 1.0%  
> Linus Walleij| 11626| 1.0%  
> Paul Mundt| 10735| 0.9%  
> Peter Zijlstra| 10202| 0.8%  
> Zhu Yi| 10197| 0.8%  
> Ben Dooks| 10150| 0.8%  
> Johannes Berg| 9532| 0.8%  
> Kalle Valo| 9263| 0.8%  
  
Ingo Molnar always shows up near the top of the changeset statistics. He has, as usual, contributed work all over the core kernel and x86 architecture code, but the bulk of his work this time is in the performance counters code; most of Peter Zijlstra's contributions were also in this area. The merging of this fast-changing subsystem caused those two developers to be responsible for 5% of the patches going into the 2.6.31 release. Paul Mundt wrote a vast number of Super-H architecture patches, and Takashi Iwai contributed large numbers of ALSA patches. 

#5 on the changesets list is Bartlomiej Zolnierkiewicz, who also comes out on top in terms of the number of lines changed. He contributed a few IDE patches, despite having handed off responsibility for that subsystem, but most of his work went into the cleaning-up of Ralink wireless drivers in the staging tree. This cleanup resulted in the removal of an amazing 208,000 lines of code. Jerry Chuang added the RealTek RTL8192SU wireless driver (to staging), Forest Bond added the VIA Technologies VT6655 driver (to staging), David Daney did a bunch of MIPS work (including adding the Octeon Ethernet driver to the staging tree), and Jerome Glisse added kernel mode setting support for Radeon graphics chipsets. 

As we have seen in the past few development cycles, the staging tree is the source of much of the change in the kernel tree. The nature of that change is, itself, changing, though. The rush of adding out-of-tree drivers to the staging tree has slowed considerably; we're starting to see more work dedicated to fixing up the code which is already there. 

The developers contributing to 2.6.31 were supported by a minimum of 194 employers. The most active of those were: 

> Most active 2.6.31 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1704| 16.0%  
> Red Hat| 1587| 14.9%  
> Intel| 878| 8.2%  
> (Unknown)| 846| 7.9%  
> IBM| 667| 6.3%  
> Novell| 614| 5.8%  
> Renesas Technology| 345| 3.2%  
> Fujitsu| 223| 2.1%  
> (Consultant)| 212| 2.0%  
> Analog Devices| 212| 2.0%  
> Oracle| 175| 1.6%  
> Nokia| 131| 1.2%  
> AMD| 129| 1.2%  
> Atheros Communications| 118| 1.1%  
> MontaVista| 104| 1.0%  
> Xelerated AB| 100| 0.9%  
> (Academia)| 92| 0.9%  
> NetApp| 91| 0.9%  
> HP| 86| 0.8%  
> Wolfson Microelectronics| 85| 0.8%  
> | By lines changed  
> ---  
> (None)| 311803| 25.8%  
> Red Hat| 124831| 10.3%  
> Realtek| 78441| 6.5%  
> Intel| 62559| 5.2%  
> Broadcom| 51806| 4.3%  
> Logic Supply| 51401| 4.3%  
> (Unknown)| 47165| 3.9%  
> Cavium Networks| 40086| 3.3%  
> IBM| 39991| 3.3%  
> Novell| 31979| 2.6%  
> Renesas Technology| 31674| 2.6%  
> (Consultant)| 23659| 2.0%  
> Emulex| 19209| 1.6%  
> University of Virginia| 17607| 1.5%  
> Nokia| 16234| 1.3%  
> Embedded Alley Solutions| 15229| 1.3%  
> Creative Technology| 12741| 1.1%  
> Oracle| 11704| 1.0%  
> Analog Devices| 10760| 0.9%  
> Texas Instruments| 10639| 0.9%  
  
The top group in either category is developers working on their own time, followed by Red Hat, which merged a few large chunks of code this time around. 

A look at non-author signoffs (a hint as to which subsystem maintainers admitted the patches into the mainline) shows a continuation of recent trends: 

> Top non-author signoffs in 2.6.31  
> ---  
> | Individuals  
> ---  
> David S. Miller| 964| 10.1%  
> Ingo Molnar| 948| 9.9%  
> Greg Kroah-Hartman| 582| 6.1%  
> John W. Linville| 575| 6.0%  
> Andrew Morton| 569| 6.0%  
> Mauro Carvalho Chehab| 535| 5.6%  
> Linus Torvalds| 254| 2.7%  
> James Bottomley| 237| 2.5%  
> Benny Halevy| 191| 2.0%  
> Paul Mundt| 159| 1.7%  
> | Employers  
> ---  
> Red Hat| 3686| 38.7%  
> Novell| 1061| 11.1%  
> Intel| 829| 8.7%  
> Google| 572| 6.0%  
> (None)| 422| 4.4%  
> IBM| 383| 4.0%  
> Linux Foundation| 254| 2.7%  
> Oracle| 228| 2.4%  
> Panasas| 193| 2.0%  
> (Consultant)| 168| 1.8%  
  
49.8% of the patches going into the mainline for 2.6.31 passed through the hands of developers working for just two companies: Red Hat and Novell. Linux kernel developers work for a large number of companies, but subsystem maintainers are increasingly concentrated in a very small number of places. 

In summary, it is a fairly typical development cycle for the kernel in recent times. The number of changes is high (but not a record), as is the number of developers. The transient effect of the staging tree is beginning to fade; it is becoming just another path for drivers heading into the mainline. As a whole, the process seems to be functioning in a smooth and robust manner. 

(As always, your editor would like to thank Greg Kroah-Hartman for his assistance in the preparation of these statistics.)
