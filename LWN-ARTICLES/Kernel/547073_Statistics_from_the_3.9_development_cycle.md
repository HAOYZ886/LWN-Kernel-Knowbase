---
title: Statistics from the 3.9 development cycle
url: https://lwn.net/Articles/547073/
date: "April 17, 2013"
category: "Releases-3.9"
author: "By Jonathan Corbet April 17, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 17, 2013

As of this writing, Linus has stated that 3.9-rc7 should be the last prepatch for the 3.9 development cycle. If that prediction holds, the final 3.9 release can be expected sometime around April 21, after a 62-day development cycle. That is not the shortest cycle ever, but it is getting close; in general, the community has been producing kernels more quickly in the last year, with no kernel after 3.3 taking more than 71 days. No kernel has gone past -rc8 since the release of [3.1-rc10](<https://lwn.net/Articles/463493/>) in October 2011 — and that was a special case caused by the kernel.org breakin. At this point, everybody seems to know how the process works, and things go pretty smoothly. 

3.8 was the most active development cycle ever. At 11,746 non-merge changesets (as of this writing), 3.9 will not beat that record, but it will set one of its own: the 1,364 developers who contributed to this kernel are the most ever. The most active of those developers were: 

> Most active 3.9 developers  
> ---  
> | By changesets  
> ---  
> Takashi Iwai| 265| 2.3%  
> H Hartley Sweeten| 259| 2.2%  
> Al Viro| 208| 1.8%  
> Tejun Heo| 186| 1.6%  
> Johannes Berg| 178| 1.5%  
> Kees Cook| 177| 1.5%  
> Daniel Vetter| 128| 1.1%  
> Alex Elder| 119| 1.0%  
> Eric W. Biederman| 109| 0.9%  
> Laurent Pinchart| 109| 0.9%  
> Mark Brown| 107| 0.9%  
> Yinghai Lu| 98| 0.8%  
> Peter Huewe| 95| 0.8%  
> Kevin McKinney| 95| 0.8%  
> Vineet Gupta| 94| 0.8%  
> Rafael J. Wysocki| 90| 0.8%  
> Hideaki Yoshifuji| 85| 0.7%  
> Jingoo Han| 81| 0.7%  
> Sachin Kamat| 76| 0.7%  
> Mauro Carvalho Chehab| 75| 0.6%  
> | By changed lines  
> ---  
> Paul Gortmaker| 34927| 4.7%  
> Laurent Pinchart| 32137| 4.3%  
> James Hogan| 27808| 3.7%  
> Johannes Berg| 25451| 3.4%  
> Takashi Iwai| 20096| 2.7%  
> Vineet Gupta| 19886| 2.7%  
> Ralf Baechle| 15210| 2.0%  
> Manjunath Hadli| 14527| 1.9%  
> George Zhang| 10154| 1.4%  
> H Hartley Sweeten| 8796| 1.2%  
> Sony Chacko| 8781| 1.2%  
> Ariel Elior| 8590| 1.1%  
> Joe Thornber| 7724| 1.0%  
> Prashant Gaikwad| 7558| 1.0%  
> Al Viro| 6749| 0.9%  
> Christoffer Dall| 6402| 0.9%  
> Andy King| 6063| 0.8%  
> Ben Skeggs| 5563| 0.7%  
> Ian Minett| 4943| 0.7%  
> Bob Moore| 4542| 0.6%  
  
H. Hartley Sweeten continues to work on the cleanup of the Comedi drivers, but, for the first time since 3.5, he has been pushed out of the top position by Takashi Iwai, who merged a vast amount of ALSA sound driver work for 3.9. Al Viro has been working on the cleanup of a number of virtual filesystem APIs, but much of his work this time around was also focused on making the signal code more generic and architecture-independent. Tejun Heo's work is divided between improving the control group subsystem, improving workqueues, and [simplifying the IDR API](<https://lwn.net/Articles/536293/>). Johannes Berg is highly active in wireless networking, and with the core mac80211 subsystem in particular. 

Paul Gortmaker got to the top of the "lines changed" column through the removal of a number of old, obsolete network drivers; the kernel lost over 34,000 lines of code as the result of his work. Laurent Pinchart did a lot of low-level embedded architecture cleanup and improvement work, and James Hogan added the new Meta architecture. 

One could look at the development statistics and conclude that the average kernel developer contributed eight or nine changesets during the 39 cycle. The truth of the matter is a little different, as can be seen in this plot: 

> ![\[Patch
counts plot\]](https://static.lwn.net/images/2013/3.9-patches.png)

Just over one third of the developers working on 3.9 contributed a single patch, and the median developer contributed two. Meanwhile, the 100 most active developers contributed more than half of all the patches merged in this cycle. This pattern where a relatively small group of developers is responsible for the bulk of the changes has not changed much in recent years. 

219 companies (that we know of) supported development of the 3.9 kernel. The most active of these companies were: 

> Most active 3.9 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1185| 10.2%  
> (None)| 1180| 10.1%  
> Red Hat| 1050| 9.0%  
> (Unknown)| 846| 7.3%  
> SUSE| 618| 5.3%  
> Google| 406| 3.5%  
> Linaro| 397| 3.4%  
> Texas Instruments| 367| 3.1%  
> IBM| 339| 2.9%  
> Samsung| 334| 2.9%  
> Vision Engraving Systems| 259| 2.2%  
> NVidia| 208| 1.8%  
> Renesas Electronics| 203| 1.7%  
> Oracle| 170| 1.5%  
> Fujitsu| 161| 1.4%  
> Broadcom| 157| 1.3%  
> Wolfson Microelectronics| 129| 1.1%  
> Inktank Storage| 128| 1.1%  
> Freescale| 119| 1.0%  
> Arista Networks| 109| 0.9%  
> | By lines changed  
> ---  
> Intel| 75386| 10.1%  
> Renesas Electronics| 66290| 8.8%  
> Wind River| 50740| 6.8%  
> Red Hat| 48424| 6.5%  
> (None)| 38479| 5.1%  
> SUSE| 38361| 5.1%  
> (Unknown)| 32336| 4.3%  
> Texas Instruments| 32333| 4.3%  
> Imagination Technologies| 27883| 3.7%  
> NVidia| 26935| 3.6%  
> Synopsys| 20298| 2.7%  
> Samsung| 19555| 2.6%  
> Broadcom| 17755| 2.4%  
> VMWare| 16332| 2.2%  
> IBM| 16313| 2.2%  
> Linaro| 13794| 1.8%  
> QLogic| 11460| 1.5%  
> Vision Engraving Systems| 10731| 1.4%  
> Google| 10581| 1.4%  
> Marvell| 8210| 1.1%  
  
For the first time ever, Intel finds itself at the top of the chart in both columns, displacing Red Hat and even exceeding the total of contributions from volunteers (those marked as "(None)" above); chances are, though, that if all the developers in the "unknown" category were known, they would push the volunteer group back to the top of the list. In general, the percentage of contributions from volunteers continues its slow decline. In today's job market, it seems, anybody who is able to get code into the kernel has to be fairly determined to reject job offers to remain a volunteer. 

In summary, the kernel development community remains healthy and vibrant, delivering vast amounts of work to Linux users via a process that appears to run like a well-oiled machine. There are very few projects, either free or proprietary, that can sustain this kind of pace for years at a time. Given the kernel's history, it seems likely that things will continue in this vein for some time; it is going to be fun to watch.
