---
title: Some numbers from the 4.19 development cycle
url: https://lwn.net/Articles/767635/
date: "October 7, 2018"
category: "Releases-4.19"
author: "By Jonathan Corbet October 7, 2018"
---

By **Jonathan Corbet**  
October 7, 2018

The [release](<https://lwn.net/Articles/767270/>) of 4.19-rc6 on September 30 is an indication that the 4.19 development cycle is heading toward its conclusion. Naturally, that means it's time to have a look at where the contributions for this cycle came from. The upheavals currently playing out in the kernel community do not show at this level, but there are some new faces to be seen in the top contributors this time around. 

As of this writing, 13,657 non-merge changesets have found their way into the mainline for 4.19. That makes this development cycle the busiest since 4.15, but only by a little bit; the patch volume in recent cycles has been remarkably constant: 

> Cycle| Changesets  
> ---|---  
> 4.15| 14,866|   
> 4.16| 13,630|   
> 4.17| 13,541|   
> 4.18| 13,283|   
> 4.19| 13,657| (so far)  
  
The changes in 4.19 were contributed by 1,710 developers, again a typical number; 253 of those developers were contributing to the kernel for the first time. The last two development cycles both removed more lines of code from the kernel than they added; that trend has come to a screeching halt in 4.19, which added 307,000 lines. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The most active 4.19 developers were: 

> Most active 4.19 developers  
> ---  
> | By changesets  
> ---  
> John Whitmore| 222| 1.6%  
> Chris Wilson| 208| 1.5%  
> Gustavo A. R. Silva| 205| 1.5%  
> Colin Ian King| 178| 1.3%  
> Arnd Bergmann| 155| 1.1%  
> Christoph Hellwig| 132| 1.0%  
> Takashi Iwai| 124| 0.9%  
> Todd Poynor| 116| 0.8%  
> Bart Van Assche| 110| 0.8%  
> Ville Syrjälä| 104| 0.8%  
> Paul E. McKenney| 101| 0.7%  
> Michael Straube| 101| 0.7%  
> Brian Foster| 90| 0.7%  
> Hans de Goede| 86| 0.6%  
> Jason Gunthorpe| 86| 0.6%  
> Boris Brezillon| 85| 0.6%  
> Geert Uytterhoeven| 83| 0.6%  
> Jerome Brunet| 79| 0.6%  
> Jakub Kicinski| 78| 0.6%  
> YueHaibing| 77| 0.6%  
> | By changed lines  
> ---  
> Jeykumar Sankaran| 32138| 4.8%  
> Richard Fitzgerald| 14390| 2.2%  
> Jason Cooper| 11415| 1.7%  
> Steven J. Hill| 10008| 1.5%  
> Stanislaw Gruszka| 8686| 1.3%  
> Darrick J. Wong| 8396| 1.3%  
> Christoph Hellwig| 8366| 1.3%  
> Simon Que| 8083| 1.2%  
> Jerome Brunet| 7702| 1.2%  
> Jiri Pirko| 6597| 1.0%  
> Gao Xiang| 6464| 1.0%  
> Jason Gunthorpe| 6333| 1.0%  
> Rob Clark| 6220| 0.9%  
> Lorenzo Bianconi| 6032| 0.9%  
> Chris Wilson| 5970| 0.9%  
> Linus Walleij| 5642| 0.9%  
> Srinivas Kandagatla| 5170| 0.8%  
> Benjamin Herrenschmidt| 5150| 0.8%  
> Jordan Crouse| 5114| 0.8%  
> David Lechner| 5063| 0.8%  
  
The ranks of the top contributors include some new names this time around. John Whitmore's work was entirely focused on improving two Realtek drivers in the staging tree. Chris Wilson made many changes to the i915 graphics driver, Gustavo A. R. Silva and Colin Ian King made small cleanups all over the tree, and Arnd Bergmann made significant fixes all over, many as part of the larger [year-2038 readiness](<https://lwn.net/Articles/717076/>) effort. 

On the "lines changed" side, Jeykumar Sankaran added a graphics driver for SDM845 chipsets. Richard Fitzgerald added support for some Cirrus Logic codecs, Jason Cooper removed the unloved Skein and Threefish crypto algorithms from the staging tree, Steven Hill cleaned up the MIPS Octeon architecture code, and Stanislaw Gruszka contributed the MediaTek mt76x0 driver. 

Work on 4.19 was supported by a minimum of 230 employers, the most active of which were: 

> Most active 4.19 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1294| 9.5%  
> (None)| 1180| 8.6%  
> Red Hat| 970| 7.1%  
> IBM| 674| 4.9%  
> (Unknown)| 662| 4.8%  
> Linaro| 604| 4.4%  
> Mellanox| 561| 4.1%  
> AMD| 546| 4.0%  
> Google| 541| 4.0%  
> SUSE| 495| 3.6%  
> Huawei Technologies| 390| 2.9%  
> (Consultant)| 309| 2.3%  
> Renesas Electronics| 294| 2.2%  
> Bootlin| 291| 2.1%  
> ARM| 272| 2.0%  
> Oracle| 250| 1.8%  
> Linux Foundation| 235| 1.7%  
> Canonical| 225| 1.6%  
> NXP Semiconductors| 204| 1.5%  
> Code Aurora Forum| 190| 1.4%  
> | By lines changed  
> ---  
> (None)| 56201| 8.5%  
> Code Aurora Forum| 53644| 8.1%  
> Intel| 52937| 8.0%  
> Red Hat| 44222| 6.7%  
> Mellanox| 35693| 5.4%  
> Linaro| 35591| 5.4%  
> IBM| 26092| 3.9%  
> Google| 24996| 3.8%  
> AMD| 20602| 3.1%  
> (Unknown)| 19136| 2.9%  
> Huawei Technologies| 17230| 2.6%  
> (Consultant)| 16199| 2.4%  
> Cirrus Logic| 14565| 2.2%  
> SUSE| 13685| 2.1%  
> Cavium| 13338| 2.0%  
> Oracle| 13309| 2.0%  
> BayLibre| 11854| 1.8%  
> ARM| 10897| 1.6%  
> Renesas Electronics| 10703| 1.6%  
> Facebook| 10082| 1.5%  
  
One thing that jumps out is the amount of work that came from developers who were not working for anybody else. That number is still low by long-term historic standards (it was [12% for 3.0](<https://lwn.net/Articles/451243/>), for example), but is higher than it has been in recent times. There were 116 developers known to be working on their own time, or just under 7% of the developers working on the kernel; they contributed 8.5% of the total work. 

With regard to testing and reviewing, the numbers this time around look like this: 

> Test and review credits in 4.19   
> ---  
> | Tested-by  
> ---  
> Pavel Machek| 40| 5.5%  
> Andrew Bowers| 27| 3.7%  
> Alexandre Courbot| 26| 3.6%  
> Arnaldo Carvalho de Melo| 26| 3.6%  
> Joel Stanley| 23| 3.2%  
> Shakeel Butt| 17| 2.4%  
> Neil Brown| 15| 2.1%  
> Hans de Goede| 14| 1.9%  
> Tony Brelinski| 13| 1.8%  
> Stan Johnson| 12| 1.7%  
> Jiri Kosina| 11| 1.5%  
> Song Liu| 10| 1.4%  
> Randy Dunlap| 10| 1.4%  
> Peter Rosin| 9| 1.2%  
> Matthias Kaehlcke| 7| 1.0%  
> Lucas Stach| 7| 1.0%  
> Hanjun Guo| 7| 1.0%  
> Ganapatrao Kulkarni| 7| 1.0%  
> Dave Penkler| 7| 1.0%  
> Aaron Brown| 7| 1.0%  
> | Reviewed-by  
> ---  
> Rob Herring| 185| 3.8%  
> Darrick J. Wong| 144| 2.9%  
> Christoph Hellwig| 134| 2.7%  
> Christian König| 101| 2.1%  
> Andrew Morton| 91| 1.8%  
> Alex Deucher| 85| 1.7%  
> Geert Uytterhoeven| 77| 1.6%  
> Simon Horman| 66| 1.3%  
> David Sterba| 63| 1.3%  
> Boris Brezillon| 62| 1.3%  
> Tony Cheng| 61| 1.2%  
> Andy Shevchenko| 57| 1.2%  
> Tvrtko Ursulin| 57| 1.2%  
> Daniel Vetter| 53| 1.1%  
> Quentin Monnet| 53| 1.1%  
> Ville Syrjälä| 52| 1.1%  
> Rodrigo Vivi| 49| 1.0%  
> Harry Wentland| 49| 1.0%  
> Fabio Estevam| 49| 1.0%  
  
Of the 13,657 patches merged for 4.19, 659 carried Tested-by tags — about 5% of the total. 4,085 (30%) carried Reviewed-by tags. The following chart shows the trends in the use of these tags in recent years: 

> ![test/review tag usage
chart](https://static.lwn.net/images/2018/trstats-4.19.svg)

As can be seen in this plot, the percentage of patches carrying Tested-by tags is at best flat over the 4.x series (which began in 2015). If one looks further back to 3.1 in late 2011, though, 3.6% of patches carried those tags, so there has been a bit of growth over that period. Reviewed-by tags, instead, are clearly appearing in more patches over time as more subsystems require them. That number, too, is up significantly from 3.1, where only 8.7% of patches had such tags. Kernel developers may be slow to adopt change, but some things do eventually change over time. 

What doesn't seem to change is that the kernel-development machine continues to crank out releases integrating vast amounts of change on a predictable basis. The community is in a period of change in a number of ways, but it is to be expected that this record, which has held steady for many years now, will continue into the future.
