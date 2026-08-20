---
title: A look at the 4.8 development cycle
url: https://lwn.net/Articles/701650/
date: "September 28, 2016"
category: "Releases-4.8"
author: "By Jonathan Corbet September 28, 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 28, 2016

As of this writing, the 4.8 development cycle is nearing its end. Linus has let it be known that a relatively unusual [-rc8 release candidate](<https://lwn.net/Articles/701879/>) will be required before the final release, but that still means that the cycle will only require 70 days, fitting into the usual pattern. A look at the development statistics for this release also fits the pattern about now. 

With regard to the release cycle, it has become boringly regular in recent years. The 3.8 kernel, released on February 18, 2013, came out on a Sunday, as has every subsequent release with the exception of 3.11, which was released on Monday, September 2, 2013. In these last few years, the only cycle that has taken longer than 70 days was 3.13, which required 77 days. The extra week that time around was forced by Linus's travels, rather than anything inherent in that cycle itself. Since then, every cycle has taken 63 or 70 days, with the sole exception of 3.16, which showed up in 56 (and one could quibble that it was really a 63-day cycle as well — that was the time Linus experimented with opening the merge window before the previous final release had been made). 

In this 70-day cycle, we have seen the addition of 13,253 non-merge changesets from 1,578 developers — so far; the numbers will increase slightly before the end. It is thus a busy cycle, though the record for the busiest (3.15, with 13,722 commits) remains unchallenged. Those developers grew the kernel by 350,000 lines this time around. The most active developers in this cycle were: 

> Most active 4.8 developers  
> ---  
> | By changesets  
> ---  
> Mauro Carvalho Chehab| 347| 2.6%  
> Chris Wilson| 266| 2.0%  
> Arnd Bergmann| 180| 1.4%  
> Daniel Vetter| 144| 1.1%  
> Geert Uytterhoeven| 139| 1.0%  
> Wei Yongjun| 129| 1.0%  
> Hans Verkuil| 121| 0.9%  
> Arnaldo Carvalho de Melo| 117| 0.9%  
> James Hogan| 107| 0.8%  
> Paul Gortmaker| 100| 0.8%  
> Trond Myklebust| 98| 0.7%  
> David Hildenbrand| 92| 0.7%  
> Christoph Hellwig| 90| 0.7%  
> Krzysztof Kozlowski| 88| 0.7%  
> Ville Syrjälä| 86| 0.6%  
> Daniel Lezcano| 82| 0.6%  
> Ben Dooks| 80| 0.6%  
> Linus Walleij| 76| 0.6%  
> Wolfram Sang| 75| 0.6%  
> Christian König| 75| 0.6%  
> | By changed lines  
> ---  
> Mauro Carvalho Chehab| 110741| 13.2%  
> Markus Heiser| 77196| 9.2%  
> Hans Verkuil| 17868| 2.1%  
> Wolfram Sang| 15211| 1.8%  
> Moni Shoua| 13039| 1.6%  
> Christoph Hellwig| 12535| 1.5%  
> Yuval Mintz| 12467| 1.5%  
> Jani Nikula| 12397| 1.5%  
> Chris Wilson| 11003| 1.3%  
> Darrick J. Wong| 7453| 0.9%  
> Arnaldo Carvalho de Melo| 7204| 0.9%  
> Marc Zyngier| 6514| 0.8%  
> Daniel Vetter| 6499| 0.8%  
> Megha Dey| 5844| 0.7%  
> Florian Fainelli| 5697| 0.7%  
> Krzysztof Kozlowski| 5600| 0.7%  
> Gavin Shan| 5343| 0.6%  
> Bryant G. Ly| 5019| 0.6%  
> Arnd Bergmann| 4914| 0.6%  
> Adrian Hunter| 4906| 0.6%  
  
Mauro Carvalho Chehab, the maintainer for the media subsystem, is traditionally a highly active developer. To understand his position at the top of both columns this time around, one need only to look back to the [4.8-rc1 announcement](<https://lwn.net/Articles/696633/>), where Linus said: 

The merge window has been fairly normal, although the patch itself looks somewhat unusual: over 20% of the patch is documentation updates, due to conversion of the drm and media documentation from docbook to the Sphinx doc format. 

Many of those documentation updates, part of [the transition in the kernel's formatted documentation subsystem](<https://lwn.net/Articles/692704/>), came from Mauro, who jumped on the task of converting the (considerable) media documentation with gusto. Other developers at the top of the "by changesets" column include Chris Wilson, whose work was focused on the Intel i915 driver; Arnd Bergmann who, when he's not maintaining the arm-soc subsystem, stays busy eliminating warnings from the kernel build; Daniel Vetter, an active DRM developer, and Geert Uytterhoeven, who did a lot of system-on-chip support work. 

In the "changed lines" column, Markus Heiser worked on the media document conversion — and contributed a fair amount of code to make the new documentation system work. Hans Verkuil did a lot of media driver work (including removing some unused drivers), Wolfram Sang spent time on on the ks7010 driver in the staging tree (along with maintaining the I2C subsystem), and Moni Shoua contributed a single patch adding the "RDMA over converged Ethernet" driver to the InfiniBand subsystem. 

Normally, work in the staging tree figures prominently in these statistics, but it is almost absent this time around. Indeed, only 386 patches have been applied to the staging tree in the 4.8 cycle, far less than the 916 seen in 4.7, or the 1,852 in 4.6. One might be tempted to think that the staging tree is slowing down, but that seems likely to be a temporary state of affairs. Indeed, it [appears](<https://lwn.net/Articles/701935/>) that the 4.9 development cycle will see over 2,300 staging commits for the addition of the [greybus subsystem](<https://lwn.net/Articles/648400/>) alone. 

Work on the 4.8 kernel was supported by 217 employers that we were able to identify. The most active employers this time around were: 

> Most active 4.8 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1960| 14.8%  
> Red Hat| 1143| 8.6%  
> (Unknown)| 806| 6.1%  
> (None)| 746| 5.6%  
> Linaro| 662| 5.0%  
> IBM| 654| 4.9%  
> Samsung| 637| 4.8%  
> SUSE| 338| 2.6%  
> Google| 294| 2.2%  
> AMD| 281| 2.1%  
> Oracle| 259| 2.0%  
> Texas Instruments| 258| 1.9%  
> Mellanox| 243| 1.8%  
> Renesas Electronics| 223| 1.7%  
> Broadcom| 217| 1.6%  
> ARM| 204| 1.5%  
> Huawei Technologies| 170| 1.3%  
> NVidia| 166| 1.3%  
> NXP Semiconductors| 163| 1.2%  
> (Consultant)| 157| 1.2%  
> | By lines changed  
> ---  
> Samsung| 120693| 14.4%  
> Intel| 104291| 12.4%  
> (None)| 102848| 12.3%  
> Red Hat| 48563| 5.8%  
> IBM| 42298| 5.0%  
> Mellanox| 29226| 3.5%  
> (Unknown)| 27671| 3.3%  
> Linaro| 22960| 2.7%  
> Broadcom| 18040| 2.2%  
> Cisco| 17868| 2.1%  
> MediaTek| 16292| 1.9%  
> QLogic| 15986| 1.9%  
> ARM| 14397| 1.7%  
> Renesas| 14283| 1.7%  
> (Consultant)| 14146| 1.7%  
> Free Electrons| 11227| 1.3%  
> Oracle| 10982| 1.3%  
> Texas Instruments| 9789| 1.2%  
> Google| 9534| 1.1%  
> Renesas Electronics| 9482| 1.1%  
  
The documentation work has shifted the numbers around here a bit but, for the most part, this table is as boring and unsurprising as usual. Samsung's position at the top of the "lines changed" column is, once again, the result of the formatted documentation transition. 

In summary, this would appear to be another relatively normal busy development cycle. The kernel development machine appears to continue to hum along smoothly, with no serious process problems evident at this level though, as the [recent discussion on backporting](<https://lwn.net/Articles/700530/>) showed, there are issues elsewhere in the community. Both the 4.8 kernel and the community that produce it appear to be working well.
