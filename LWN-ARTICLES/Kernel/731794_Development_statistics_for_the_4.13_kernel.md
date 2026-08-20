---
title: Development statistics for the 4.13 kernel
url: https://lwn.net/Articles/731794/
date: "August 24, 2017"
category: "Releases-4.13"
author: "By Jonathan Corbet August 24, 2017"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 24, 2017

As of this writing, the 4.13 kernel appears headed toward release on September 3, after a nine-week development cycle. It must, therefore, be about time for a look at the statistics for this development cycle. The picture that results shows a fairly typical kernel cycle with, as usual, few surprises. 

Midway between 4.13-rc6 and 4.13-rc7, 12,677 non-merge changesets had found their way into the mainline. That makes 4.13 the smallest cycle since 4.7, which finished with 12,283 changesets. Chances are, though, that this cycle will surpass 4.11 (12,724) by the time it is done. So, while there may be signs of a (northern hemisphere) summer slowdown, 4.13 remains generally comparable with its predecessors with respect to patch volume. 

1,634 developers have contributed during this cycle, a significant drop from the record set with 4.12 (1,825) but comparable with 4.10 (1,647). The most active of those developers were: 

> Most active 4.13 developers  
> ---  
> | By changesets  
> ---  
> Christoph Hellwig| 252| 2.0%  
> Mauro Carvalho Chehab| 184| 1.5%  
> Thomas Gleixner| 151| 1.2%  
> Arnd Bergmann| 138| 1.1%  
> Takashi Iwai| 134| 1.1%  
> Chris Wilson| 130| 1.0%  
> Colin Ian King| 123| 1.0%  
> Arvind Yadav| 123| 1.0%  
> Al Viro| 117| 0.9%  
> Masahiro Yamada| 113| 0.9%  
> Kuninori Morimoto| 102| 0.8%  
> Jakub Kicinski| 99| 0.8%  
> Johannes Berg| 98| 0.8%  
> Dan Carpenter| 93| 0.7%  
> Vivien Didelot| 90| 0.7%  
> Paul E. McKenney| 83| 0.7%  
> Geert Uytterhoeven| 82| 0.6%  
> Andy Shevchenko| 77| 0.6%  
> Kees Cook| 76| 0.6%  
> Nicholas Piggin| 72| 0.6%  
> | By changed lines  
> ---  
> Alex Deucher| 279567| 29.9%  
> Mauro Carvalho Chehab| 32256| 3.5%  
> Robert Bragg| 22511| 2.4%  
> Steve Longerbeam| 12486| 1.3%  
> Stanimir Varbanov| 11236| 1.2%  
> Christoph Hellwig| 10187| 1.1%  
> Michal Kalderon| 9818| 1.1%  
> Yuval Mintz| 9373| 1.0%  
> Lionel Landwerlin| 8960| 1.0%  
> Igor Mitsyanko| 8485| 0.9%  
> John Johansen| 7806| 0.8%  
> Mika Westerberg| 7004| 0.7%  
> Chris Wilson| 6723| 0.7%  
> Ben Skeggs| 6305| 0.7%  
> Hans de Goede| 5975| 0.6%  
> Geert Uytterhoeven| 5722| 0.6%  
> Gilad Ben-Yossef| 5580| 0.6%  
> Al Viro| 5478| 0.6%  
> Ilan Tayari| 5215| 0.6%  
> Serge Semin| 4978| 0.5%  
  
The top contributor of changesets this time around was Christoph Hellwig, who made significant improvements all over the filesystem and block I/O layers. Mauro Carvalho Chehab continues to be a relentless generator of patches in his role as the media subsystem maintainer; many of his changes touched the documentation directory as well. Thomas Gleixner was busy in the interrupt-handling and timer code, Arnd Bergmann (as usual) contributed fixes all over the tree, and Takashi Iwai made many changes as the maintainer of the audio subsystem. 

Once again, Alex Deucher topped the "lines changed" column by adding yet another massive set of AMD GPU register definitions. Robert Bragg, instead, added a bunch of i915 register configurations. Steve Longerbeam and Stanimir Varbanov both added media subsystem drivers. 

As has been the case in recent cycles, the developers appearing in these lists are generally not working on the staging tree. That is a significant change from a few years ago, when staging work was the source of many of the changesets going into the mainline kernel. One might almost be tempted to believe that the staging tree has done what it was meant to do, and the bulk of those out-of-tree drivers have now been merged. More likely, though, is that this is just a lull in staging work; substandard drivers are in anything but short supply. 

A minimum of 203 employers supported work on the code that was merged for 4.13, a fairly normal number (though, once again, a significant drop from 4.12, which had support from 233). The most active of those employers were: 

> Most active 4.13 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1474| 11.6%  
> (None)| 887| 7.0%  
> (Unknown)| 756| 6.0%  
> Red Hat| 750| 5.9%  
> IBM| 537| 4.2%  
> SUSE| 495| 3.9%  
> Linaro| 475| 3.7%  
> Google| 416| 3.3%  
> AMD| 410| 3.2%  
> (Consultant)| 389| 3.1%  
> Renesas Electronics| 331| 2.6%  
> Samsung| 323| 2.5%  
> Mellanox| 281| 2.2%  
> Oracle| 274| 2.2%  
> ARM| 265| 2.1%  
> Free Electrons| 232| 1.8%  
> Canonical| 203| 1.6%  
> Cavium| 201| 1.6%  
> Broadcom| 178| 1.4%  
> linutronix| 172| 1.4%  
> | By lines changed  
> ---  
> AMD| 296975| 31.8%  
> Intel| 79179| 8.5%  
> (None)| 53207| 5.7%  
> Red Hat| 40166| 4.3%  
> Samsung| 36962| 4.0%  
> Cavium| 32397| 3.5%  
> Linaro| 30870| 3.3%  
> (Unknown)| 30295| 3.2%  
> IBM| 21185| 2.3%  
> Mellanox| 19441| 2.1%  
> Renesas Electronics| 17946| 1.9%  
> (Consultant)| 14005| 1.5%  
> Free Electrons| 13043| 1.4%  
> Mentor Graphics| 12768| 1.4%  
> SUSE| 12742| 1.4%  
> Google| 12288| 1.3%  
> ARM| 11466| 1.2%  
> Texas Instruments| 10149| 1.1%  
> ST Microelectronics| 9062| 1.0%  
> Broadcom| 8945| 1.0%  
  
Once again, there are few surprises here; these lists don't change much from one cycle to the next. 

One thing we have occasionally commented on over the years is a perceived decrease in the contributions from developers working on their own time. The 887 changes known to be from volunteers in 4.13 make up 7% of the total, a relatively low number. But perhaps percentages are not the right unit here. Looking at the absolute count of changesets from volunteers since 3.0 was released in July 2011 reveals a trend like this: 

> ![\[Changesets from
volunteers\]](https://static.lwn.net/images/2017/4.13-volunteers.svg)

That plot does suggest an overall decrease in the number of patches received from developers working on their own time. But it may not be an entirely accurate picture. The table above also shows 756 changes coming from developers with unknown affiliation. There were 263 such developers participating in the 4.13 development cycle, contributing an average of just under three patches each; 165 of them contributed a single patch. One could well argue that the bulk of this group is highly likely to fit into the "volunteers" category. Some of them may well be doing kernel patches at work, but it's clearly not a significant part of their job. 

If one plots the number of changesets coming from both known volunteers and shadowy mysterious developers, the result is: 

> ![\[Changesets from
volunteers\]](https://static.lwn.net/images/2017/4.13-total.svg)

That line looks rather more level, suggesting that the number of changes contributed by volunteers has remained roughly the same over the last six years. Note that the overall changeset volume has increased significantly over this period; the 3.0 development cycle had 9,153, for example. So, while the volume of changes going into the kernel is increasing, the volume from volunteer developers cannot be said to be increasing with it — but, perhaps, it is not shrinking either. 

Overall, the kernel-development machine continues to hum along, cranking out a new kernel every nine or ten weeks. The predictability of the process may lead to relatively boring statistics articles, but predictability is a good thing in a critical low-level system component.
