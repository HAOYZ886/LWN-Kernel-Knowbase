---
title: Development statistics for the 4.2 kernel
url: https://lwn.net/Articles/654633/
date: "August 18, 2015"
category: "Releases-4.2"
author: "By Jonathan Corbet August 18, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 18, 2015

As of this writing, the [4.2-rc7](<https://lwn.net/Articles/654678/>) prepatch is out and the final 4.2 kernel looks to be (probably) on-track to be released on August 23\. Tradition says that it's time for a look at the development statistics for this cycle. 4.2, in a couple of ways, looks a bit different from recent cycles, with some older patterns reasserting themselves. 

At the end of the merge window, there was some speculation as to whether 4.2 would be the busiest development cycle yet. The current record holder is 3.15, which had 13,722 non-merge changesets at the time of its final release. 4.2, which had 13,555 at the -rc7 release, looks to fall a little short of that figure. So we will not have broken the record for the most changesets in any development cycle, but it was awfully close. 

One record that did fall, though, is the number of developers contributing code to the kernel. The previous record holder (4.1, at 1,539) didn't keep that position for long; 1,569 developers have contributed to 4.2. Of those developers, 279 have made their first contribution to the Linux kernel. An eye-opening 1.09 million lines of code were added this time around with 285,000 removed, for a total growth of 800,000 lines of code. 

The most active developers this time around were: 

> Most active 4.2 developers  
> ---  
> | By changesets  
> ---  
> Ingo Molnar| 304| 2.2%  
> Mauro Carvalho Chehab| 203| 1.5%  
> Herbert Xu| 171| 1.3%  
> Krzysztof Kozlowski| 161| 1.2%  
> Geert Uytterhoeven| 149| 1.1%  
> Al Viro| 140| 1.0%  
> Lars-Peter Clausen| 137| 1.0%  
> H Hartley Sweeten| 136| 1.0%  
> Thomas Gleixner| 127| 0.9%  
> Hans Verkuil| 124| 0.9%  
> Tejun Heo| 110| 0.8%  
> Alex Deucher| 95| 0.7%  
> Paul Gortmaker| 91| 0.7%  
> Vineet Gupta| 88| 0.7%  
> Jiang Liu| 84| 0.6%  
> Christoph Hellwig| 79| 0.6%  
> Hans de Goede| 78| 0.6%  
> Arnaldo Carvalho de Melo| 77| 0.6%  
> Mateusz Kulikowski| 74| 0.5%  
> Takashi Iwai| 73| 0.5%  
> | By changed lines  
> ---  
> Alex Deucher| 425501| 35.7%  
> Johnny Kim| 33726| 2.8%  
> Raghu Vatsavayi| 14484| 1.2%  
> Greg Kroah-Hartman| 12500| 1.0%  
> Stephen Boyd| 11062| 0.9%  
> Dan Williams| 10736| 0.9%  
> Hans Verkuil| 10641| 0.9%  
> Narsimhulu Musini| 10263| 0.9%  
> Ingo Molnar| 9254| 0.8%  
> Jakub Kicinski| 8531| 0.7%  
> Herbert Xu| 8515| 0.7%  
> Yoshinori Sato| 7612| 0.6%  
> Saeed Mahameed| 7493| 0.6%  
> Sunil Goutham| 7471| 0.6%  
> Christoph Hellwig| 7384| 0.6%  
> Vineet Gupta| 7171| 0.6%  
> Mateusz Kulikowski| 6852| 0.6%  
> Maxime Ripard| 6767| 0.6%  
> Sudeep Dutt| 6647| 0.6%  
> Mauro Carvalho Chehab| 6422| 0.5%  
  
Some years ago, Ingo Molnar routinely topped the per-changesets list, but he has been busy with other pursuits recently. That changed this time around, though, with a massive rewrite of the low-level x86 floating-point-unit management code. Mauro Carvalho Chehab continues to be an active maintainer of the media subsystem, and Herbert Xu's work almost entirely reflects his role as the maintainer of the kernel's crypto subsystem. Krzysztof Kozlowski contributed cleanups throughout the driver subsystem, and Geert Uytterhoeven, despite being the m68k architecture maintainer, did most of his work within the ARM tree and related driver subsystems. 

On the "lines added" side, Alex Deucher accounted for nearly half of the entire growth of the kernel this time around with the addition of the new amdgpu graphics driver. Johnny Kim added the wilc1000 network driver to the staging tree, Raghu Vatsavayi added support for Cavium Liquidio Ethernet adapters, Greg Kroah-Hartman removed the obsolete i2o subsystem, and Stephen Boyd removed a bunch of old driver code while adding driver support for QCOM SPMI regulators and more. 

The top contributor statistics in recent years have often been dominated by developers generating lots of cleanup patches or reworking staging drivers. One might expect to see a lot of that activity in an especially busy development cycle, but that is not the case for 4.2. Instead, the top contributors include many familiar names and core contributors. One might be tempted to think that the cleanup work is finally approaching completion, but one would be highly likely to be disappointed in future development cycles. 

The most active companies supporting development in the 4.2 cycle (of 236 total) were: 

> Most active 4.2 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1665| 12.3%  
> Red Hat| 1639| 12.1%  
> (Unknown)| 884| 6.5%  
> (None)| 884| 6.5%  
> Samsung| 681| 5.0%  
> SUSE| 496| 3.7%  
> Linaro| 449| 3.3%  
> (Consultant)| 412| 3.0%  
> IBM| 391| 2.9%  
> AMD| 286| 2.1%  
> Google| 246| 1.8%  
> Renesas Electronics| 203| 1.5%  
> Free Electrons| 203| 1.5%  
> Texas Instruments| 191| 1.4%  
> Facebook| 176| 1.3%  
> Oracle| 163| 1.2%  
> Freescale| 156| 1.2%  
> ARM| 145| 1.1%  
> Cisco| 142| 1.0%  
> Broadcom| 138| 1.0%  
> | By lines changed  
> ---  
> AMD| 438094| 36.8%  
> Intel| 96331| 8.1%  
> Red Hat| 62959| 5.3%  
> (None)| 46140| 3.9%  
> (Unknown)| 41886| 3.5%  
> Atmel| 34942| 2.9%  
> Samsung| 29326| 2.5%  
> Linaro| 22714| 1.9%  
> Cisco| 21170| 1.8%  
> SUSE| 18891| 1.6%  
> Code Aurora Forum| 18435| 1.5%  
> Mellanox| 18044| 1.5%  
> (Consultant)| 15234| 1.3%  
> IBM| 15095| 1.3%  
> Cavium Networks| 14580| 1.2%  
> Free Electrons| 13640| 1.1%  
> Unisys| 13428| 1.1%  
> Linux Foundation| 12617| 1.1%  
> MediaTek| 11856| 1.0%  
> Google| 11811| 1.0%  
  
Once again, there are few surprises here. At 6.5%, the percentage of changes coming from volunteers is at its lowest point ever. AMD, unsurprisingly, dominated the lines-changed column with the addition of the amdgpu driver. Beyond that, it is mostly the usual companies supporting kernel development in the usual way. 

The kernel community depends heavily on its testers and bug reporters; at least some of the time, their contribution is recorded as Tested-by and Reported-by tags in the patches themselves. In the 4.2 development cycle, 946 Tested-by credits were placed in 729 patches, and 611 Reported-by credits were placed in 682 patches. The most active contributors in this area were: 

> Most active 4.2 testers and reporters  
> ---  
> | Tested-by credits  
> ---  
> Joerg Roedel| 40| 4.2%  
> Keita Kobayashi| 35| 3.7%  
> Krishneil Singh| 31| 3.3%  
> Arnaldo Carvalho de Melo| 30| 3.2%  
> Ira Weiny| 24| 2.5%  
> Doug Ledford| 23| 2.4%  
> Alex Ng| 22| 2.3%  
> Aaron Brown| 21| 2.2%  
> Javier Martinez Canillas| 19| 2.0%  
> ZhenHua Li| 19| 2.0%  
> | Reported-by credits  
> ---  
> Wu Fengguang| 76| 11.1%  
> Dan Carpenter| 41| 6.0%  
> Russell King| 23| 3.4%  
> Ingo Molnar| 12| 1.8%  
> Stephen Rothwell| 10| 1.5%  
> Linus Torvalds| 8| 1.2%  
> Hartmut Knaack| 7| 1.0%  
> Huang Ying| 6| 0.9%  
> Christoph Hellwig| 5| 0.7%  
> Sudeep Holla| 5| 0.7%  
  
The power of Wu Fengguang's zero-day build robot can be seen here; it resulted in 11% of all of the credited bug reports in this development cycle. The work of all of the kernel's testers and bug reporters leads to a more stable kernel release for everybody. The biggest concern with these numbers, perhaps, is that we might still not be doing a thorough job of documenting the contribution of all of our testers and reporters. 

All told, the kernel development community continues to run like a well-tuned machine, producing stable kernel releases on a predictable (and fast) schedule. Back in 2010, your editor [worried](<https://lwn.net/Articles/393694/>) that the community might be headed toward another scalability crisis, but such worries have proved to be unfounded, for now at least. There must certainly be limits to the volume of change that can be managed by the current development model, but we do not appear to have reached them yet.
