---
title: Contribution statistics for the 3.10 development cycle
url: https://lwn.net/Articles/555968/
date: "June 26, 2013"
category: "Releases-3.10"
author: "By Jonathan Corbet June 26, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 26, 2013

The 3.10 kernel development cycle is nearing its completion; as of this writing, the [3.10-rc7 prepatch](<https://lwn.net/Articles/556033/>) is out and the kernel appears to be stabilizing as expected. As predicted, 3.10 has turned out to be the busiest development cycle ever, with almost 13,500 non-merge changesets pulled into the mainline repository (so far). What follows is LWN's traditional look at where those changes came from. 

3.9 set a record of its own, with 1,388 developers contributing changes. So far, with a mere 1,374 contributors, 3.10 falls short of that record, but that situation clearly could change before the final release is made. The size of our development community, it seems, continues to increase. 

The most active 3.10 developers were: 

> Most active 3.10 developers  
> ---  
> | By changesets  
> ---  
> H Hartley Sweeten| 392| 2.9%  
> Jingoo Han| 299| 2.2%  
> Hans Verkuil| 293| 2.2%  
> Alex Elder| 268| 2.0%  
> Al Viro| 205| 1.5%  
> Felipe Balbi| 202| 1.5%  
> Sachin Kamat| 192| 1.4%  
> Laurent Pinchart| 174| 1.3%  
> Johan Hovold| 159| 1.2%  
> Mauro Carvalho Chehab| 158| 1.2%  
> Wei Yongjun| 139| 1.0%  
> Arnd Bergmann| 138| 1.0%  
> Eduardo Valentin| 138| 1.0%  
> Axel Lin| 112| 0.8%  
> Lee Jones| 111| 0.8%  
> Lars-Peter Clausen| 99| 0.7%  
> Kuninori Morimoto| 98| 0.7%  
> Tejun Heo| 97| 0.7%  
> Mark Brown| 97| 0.7%  
> Johannes Berg| 96| 0.7%  
> | By changed lines  
> ---  
> Joe Perches| 34561| 4.5%  
> Hans Verkuil| 18739| 2.4%  
> Kent Overstreet| 18690| 2.4%  
> Larry Finger| 17222| 2.2%  
> Greg Kroah-Hartman| 16610| 2.2%  
> Shawn Guo| 12879| 1.7%  
> Dave Chinner| 12838| 1.7%  
> Paul Zimmerman| 12637| 1.6%  
> H Hartley Sweeten| 12518| 1.6%  
> Al Viro| 11116| 1.4%  
> Andrey Smirnov| 11107| 1.4%  
> Mauro Carvalho Chehab| 9726| 1.3%  
> Laurent Pinchart| 9258| 1.2%  
> Jussi Kivilinna| 8960| 1.2%  
> Lee Jones| 8598| 1.1%  
> Sylwester Nawrocki| 8305| 1.1%  
> Artem Bityutskiy| 8094| 1.0%  
> Dave Airlie| 7546| 1.0%  
> Guenter Roeck| 7510| 1.0%  
> Sanjay Lal| 7428| 1.0%  
  
H. Hartley Sweeten's position at the top of the list seems like a permanent aspect of these reports as he continues his work on the endless task of cleaning up the Comedi drivers in the staging tree. Jingoo Han contributed a long list of driver cleanup patches, moving the code toward the use of standard helper functions and the "managed" resource allocation API. Hans Verkuil improved a number of video acquisition drivers as part of his new(ish) role as the maintainer of the Video4Linux subsystem. Alex Elder's work is focused on the Ceph filesystem and associated "RADOS" block device, and Al Viro implemented a large number of core kernel improvements and API changes. Together, these five developers accounted for nearly 11% of all the changes going into the kernel. 

In the "lines changed" column, Joe Perches topped the list with a set of patches effecting whitespace cleanups, `printk()` format changes, `checkpatch.pl` tweaks, and more. Kent Overstreet added the [bcache](<https://lwn.net/Articles/497024/>) block caching subsystem and a number of asynchronous I/O improvements. Larry Finger's 17 patches added new features and device support to the rtlwifi driver, and Greg Kroah-Hartman removed the Android "CCG" USB gadget driver from the staging tree. 

Just over 200 employers are known to have supported work on the 3.10 kernel. The most active of these were: 

> Most active 3.10 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1495| 11.1%  
> Red Hat| 1269| 9.4%  
> Intel| 912| 6.8%  
> Linaro| 877| 6.5%  
> Texas Instruments| 765| 5.7%  
> (Unknown)| 746| 5.5%  
> Samsung| 615| 4.6%  
> IBM| 402| 3.0%  
> Vision Engraving Systems| 392| 2.9%  
> Google| 350| 2.6%  
> SUSE| 332| 2.5%  
> Renesas Electronics| 331| 2.5%  
> Cisco| 300| 2.2%  
> Inktank Storage| 277| 2.1%  
> Broadcom| 182| 1.3%  
> NVidia| 180| 1.3%  
> Freescale| 175| 1.3%  
> Oracle| 175| 1.3%  
> Trend Micro| 139| 1.0%  
> Fujitsu| 138| 1.0%  
> | By lines changed  
> ---  
> (None)| 118326| 15.3%  
> Red Hat| 88080| 11.4%  
> Linaro| 64697| 8.4%  
> Intel| 50641| 6.6%  
> Google| 33342| 4.3%  
> Cisco| 24109| 3.1%  
> (Unknown)| 24033| 3.1%  
> Samsung| 20893| 2.7%  
> Texas Instruments| 20289| 2.6%  
> NVidia| 18470| 2.4%  
> Linux Foundation| 16759| 2.2%  
> Renesas Electronics| 15777| 2.0%  
> IBM| 14385| 1.9%  
> QLogic| 14165| 1.8%  
> Synopsys| 13698| 1.8%  
> Vision Engraving Systems| 13111| 1.7%  
> Broadcom| 12770| 1.7%  
> Synapse Product Development| 11107| 1.4%  
> OpenSource AB| 9584| 1.2%  
> SUSE| 9479| 1.2%  
  
With 3.10, Red Hat regained its usual place as the company with the most contributions, though even Red Hat, once again, falls short of the contributions from volunteers. The increase in contributions from the mobile and embedded community continues its impressive growth; Linaro, in particular, continues to grow, with 42 developers contributing code under its name to 3.10. 

In summary, the kernel's busiest development cycle ever shows the continuation of a number of patterns that have been observed for a while: increasing participation from the mobile and embedded worlds, more developers, and more companies. There was a slight uptick in volunteer contributors this time around, but it is not at all clear that the long-term decline in that area has been interrupted. As a whole, the kernel development machine continues to operate in its familiar, predictable, and productive manner.
