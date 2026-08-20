---
title: Development statistics for 4.9
url: https://lwn.net/Articles/708266/
date: "December 7, 2016"
category: "Releases-4.9"
author: "By Jonathan Corbet December 7, 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 7, 2016

The 4.9 development cycle, expected to come to a close on December 11, will be by far the busiest cycle in the history of the kernel project. A look at the repository activity for this release, though, shows that it is, in many ways, just another typical cycle for the kernel community with one large exceptional factor. 

As of this writing, 16,150 non-merge changesets have been pulled into the mainline repository for 4.9. That far exceeds the 13,722 changesets pulled for 3.15 which was, until now, the busiest cycle ever. There is no doubt that the merging of the [Greybus driver](<https://lwn.net/Articles/648400/>) had a lot to do with that but, even if one subtracts the 2,388 changesets associated with Greybus, 4.9 would still (just barely) set a new record. Clearly there has been a lot going on in the last few months. 

The changes in 4.9 were contributed by a total of 1,719 developers — also a new record. The most active of those developers were: 

> Most active 4.9 developers  
> ---  
> | By changesets  
> ---  
> Johan Hovold| 599| 3.7%  
> Viresh Kumar| 421| 2.6%  
> Greg Kroah-Hartman| 376| 2.3%  
> Alex Elder| 345| 2.1%  
> Chris Wilson| 335| 2.1%  
> Mauro Carvalho Chehab| 238| 1.5%  
> Kuninori Morimoto| 204| 1.3%  
> Wolfram Sang| 183| 1.1%  
> Wei Yongjun| 175| 1.1%  
> David Howells| 149| 0.9%  
> Arnd Bergmann| 135| 0.8%  
> Bryan O'Donoghue| 134| 0.8%  
> Ivan Safonov| 134| 0.8%  
> Sergio Paracuellos| 120| 0.7%  
> Bjorn Helgaas| 117| 0.7%  
> Javier Martinez Canillas| 108| 0.7%  
> Alex Deucher| 107| 0.7%  
> Markus Elfring| 106| 0.7%  
> Colin Ian King| 105| 0.7%  
> Linus Walleij| 95| 0.6%  
> | By changed lines  
> ---  
> Jes Sorensen| 66930| 7.0%  
> Rex Zhu| 58288| 6.1%  
> Laurent Pinchart| 33602| 3.5%  
> Greg Kroah-Hartman| 32002| 3.3%  
> Mauro Carvalho Chehab| 14747| 1.5%  
> Chris Wilson| 14661| 1.5%  
> Ken Wang| 14556| 1.5%  
> Viresh Kumar| 13389| 1.4%  
> Dom Cobley| 12816| 1.3%  
> Christoph Hellwig| 12459| 1.3%  
> Darrick J. Wong| 11646| 1.2%  
> Hans Verkuil| 11480| 1.2%  
> Johan Hovold| 11465| 1.2%  
> Ram Amrani| 11318| 1.2%  
> Alex Elder| 11060| 1.2%  
> David Howells| 11045| 1.2%  
> Netanel Belgazal| 10858| 1.1%  
> Lijun Ou| 10849| 1.1%  
> Huang Rui| 10157| 1.1%  
> Maruthi Srinivas Bayyavarapu| 10092| 1.1%  
  
The top four developers of the Greybus code, Johan Hovold, Viresh Kumar, Greg Kroah-Hartman, and Alex Elder, ended up as the top four changeset contributors overall. Between the four of them, they contributed 1,741 changesets in 4.9. The fifth developer in the by-changesets column, Chris Wilson, worked on the i915 graphics driver. 

In the "lines changed" column, Jes Sorensen removed the rtl8723au network driver, which has been superseded by the rtl8xxxu driver. Rex Zhu worked on the AMD "PowerPlay" graphics driver, Laurent Pinchart worked on various media drivers (including the Greybus camera driver), Greg Kroah-Hartman added a lot of Greybus code, and Mauro Carvalho Chehab did a lot of work as the media subsystem maintainer and in support of the ongoing documentation transition to Sphinx. 

228 companies (that we know about) supported work on the 4.9 kernel. The most active of those employers were: 

> Most active 4.9 employers  
> ---  
> | By changesets  
> ---  
> Linaro| 1876| 11.6%  
> Intel| 1869| 11.6%  
> (Unknown)| 1293| 8.0%  
> (Consultant)| 1055| 6.5%  
> (None)| 924| 5.7%  
> Red Hat| 916| 5.7%  
> Google| 541| 3.3%  
> Samsung| 535| 3.3%  
> AMD| 476| 2.9%  
> IBM| 401| 2.5%  
> Renesas Electronics| 331| 2.0%  
> Huawei Technologies| 274| 1.7%  
> Linux Foundation| 262| 1.6%  
> Mellanox| 237| 1.5%  
> SUSE| 233| 1.4%  
> ARM| 228| 1.4%  
> Oracle| 207| 1.3%  
> Texas Instruments| 167| 1.0%  
> BayLibre| 146| 0.9%  
> Broadcom| 136| 0.8%  
> | By lines changed  
> ---  
> AMD| 105820| 11.1%  
> Red Hat| 104492| 10.9%  
> Intel| 89456| 9.4%  
> Linaro| 75310| 7.9%  
> (Consultant)| 61718| 6.5%  
> (Unknown)| 45676| 4.8%  
> (None)| 34464| 3.6%  
> Google| 31054| 3.2%  
> Samsung| 25438| 2.7%  
> Oracle| 14425| 1.5%  
> Huawei Technologies| 14220| 1.5%  
> IBM| 13432| 1.4%  
> Raspberry PI| 12816| 1.3%  
> QLogic| 12439| 1.3%  
> Mellanox| 12284| 1.3%  
> Cavium Networks| 11741| 1.2%  
> NXP Semiconductors| 11662| 1.2%  
> Cisco| 11507| 1.2%  
> Linux Foundation| 11182| 1.2%  
> Amazon| 10858| 1.1%  
  
Linaro made it to the top of the "by changesets" column this time around by virtue of work on — wait for it — Greybus. Otherwise there is not a lot that is surprising in this table. As usual, quite a few companies work to support Linux, but the list of the most active of those tends not to change much from one cycle to the next. 

Finally, it is worth noting that 301 developers made their first contribution to the kernel during the 4.9 development cycle. That, too, is arguably a record ("arguably" because 2.6.25 had 332 new developers but, since the tracking only begins with the Git era at 2.6.12, some of those "new" developers had certainly made contributions before). Let's look at first-time contributors since 3.0: 

> ![\[First-time contributors\]](https://static.lwn.net/images/2016/firsttime-4.9.svg)

The plot shows a slight but observable upward slope, especially since the recent-period low of 183 during the 3.5 development cycle; the number of new contributors coming into the kernel community is growing over time. For all its faults, the kernel community is still able to attract new developers and, thus, continues to grow. That suggests that the pace of kernel development is unlikely to slow much in the near future.
