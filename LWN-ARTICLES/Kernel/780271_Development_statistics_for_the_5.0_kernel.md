---
title: Development statistics for the 5.0 kernel
url: https://lwn.net/Articles/780271/
date: "February 21, 2019"
category: "Releases-5.0"
author: "By Jonathan Corbet February 21, 2019"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 21, 2019

The announcement of the [5.0-rc7](<https://lwn.net/Articles/780019/>) kernel prepatch on February 17 signaled the imminent release of the final 5.0 kernel and the end of this development cycle. 5.0, as it turns out, brought in fewer changesets than its immediate predecessors, but it was still a busy cycle with a lot of developers participating. Read on for an overview of where the work came from in this release cycle. 

As of this writing, 12,517 non-merge changesets have been pulled into the mainline repository for the 5.0 release. This is low compared to the kernels that came before: 

> Cycle| Changesets  
> ---|---  
> 4.15| 14,866|   
> 4.16| 13,630|   
> 4.17| 13,541|   
> 4.18| 13,283|   
> 4.19| 14,043|   
> 4.20| 13,884|   
> 5.0| 12,517| (so far)  
  
One has to go back to 4.7, released in July 2016, to find a development cycle that brought in fewer changesets than 5.0. The number of developers contributing to 5.0 was 1,712, roughly equivalent to previous cycles; 276 of those developers made their first kernel contribution in this development cycle. 

The most active developers were: 

> Most active 5.0 developers  
> ---  
> | By changesets  
> ---  
> Christoph Hellwig| 213| 1.7%  
> Masahiro Yamada| 135| 1.1%  
> Colin Ian King| 135| 1.1%  
> Jens Axboe| 112| 0.9%  
> Arnaldo Carvalho de Melo| 112| 0.9%  
> Yangtao Li| 106| 0.8%  
> Yue Haibing| 100| 0.8%  
> Kuninori Morimoto| 95| 0.8%  
> Andy Shevchenko| 94| 0.8%  
> Rob Herring| 92| 0.7%  
> Maxime Ripard| 91| 0.7%  
> Boris Brezillon| 89| 0.7%  
> Jakub Kicinski| 83| 0.7%  
> Michael Straube| 83| 0.7%  
> Thierry Reding| 82| 0.7%  
> Ville Syrjälä| 82| 0.7%  
> Geert Uytterhoeven| 80| 0.6%  
> Linus Walleij| 80| 0.6%  
> Paul E. McKenney| 78| 0.6%  
> Gustavo A. R. Silva| 77| 0.6%  
> | By changed lines  
> ---  
> Olof Johansson| 41834| 6.0%  
> Kan Liang| 31458| 4.5%  
> Yong Zhi| 22799| 3.3%  
> Aaro Koskinen| 20462| 3.0%  
> Firoz Khan| 15981| 2.3%  
> Jens Axboe| 13009| 1.9%  
> Tony Lindgren| 12237| 1.8%  
> Boris Brezillon| 11422| 1.7%  
> Sean Christopherson| 10614| 1.5%  
> Dong Aisheng| 7998| 1.2%  
> Eric Biggers| 7476| 1.1%  
> Manivannan Sadhasivam| 6724| 1.0%  
> Christoph Hellwig| 6199| 0.9%  
> Federico Vaga| 5877| 0.8%  
> Jordan Crouse| 5772| 0.8%  
> Kuninori Morimoto| 5255| 0.8%  
> Florian Westphal| 5120| 0.7%  
> Mauro Carvalho Chehab| 5097| 0.7%  
> Lorenzo Bianconi| 4941| 0.7%  
> Sagi Grimberg| 4827| 0.7%  
  
Christoph Hellwig was the most prolific contributor of changesets this time around; he did a lot of work in the block subsystem and the DMA API. Masahiro Yamada's work was mostly focused on improvements to the kernel's build system, Colin Ian King continues to make spelling and coding-style fixes throughout the tree, Jens Axboe converted a lot of block drivers to the multiqueue API (along with many other block-layer changes), and Arnaldo Carvalho de Melo worked extensively on the `perf` utility. Of the top twenty developers with regard to changesets, only one got there through work on the staging tree — a significant change from years past. 

Switching to the "lines changed" column: Olof Johansson only contributed seven changesets to 5.0, but one of them was removing the old and unmaintained eicon ISDN driver. Other top contributors in the "lines changed" column include Kan Liang for adding some JSON metrics to the `perf` utility, Yong Zhi for the Intel IPU3 driver, Aaro Koskinen for work on MIPS OCTEON support, and Firoz Khan, who reworked how the system-call tables are generated for most architectures. 

A total of 226 employers supported work on 5.0, which is a typical number. The most active of those were: 

> Most active 5.0 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1360| 10.9%  
> (None)| 937| 7.5%  
> (Unknown)| 867| 6.9%  
> Red Hat| 859| 6.9%  
> Linaro| 552| 4.4%  
> Google| 487| 3.9%  
> Mellanox| 481| 3.8%  
> SUSE| 418| 3.3%  
> AMD| 415| 3.3%  
> Renesas Electronics| 394| 3.1%  
> IBM| 347| 2.8%  
> Huawei Technologies| 320| 2.6%  
> (Consultant)| 311| 2.5%  
> Facebook| 268| 2.1%  
> Bootlin| 261| 2.1%  
> NXP Semiconductors| 247| 2.0%  
> ARM| 226| 1.8%  
> Oracle| 204| 1.6%  
> Canonical| 171| 1.4%  
> Code Aurora Forum| 148| 1.2%  
> | By lines changed  
> ---  
> Intel| 116158| 16.8%  
> Facebook| 66816| 9.7%  
> Linaro| 40368| 5.8%  
> Red Hat| 33041| 4.8%  
> (None)| 32191| 4.7%  
> (Unknown)| 26858| 3.9%  
> Mellanox| 26487| 3.8%  
> Google| 24099| 3.5%  
> Nokia| 20600| 3.0%  
> Bootlin| 19019| 2.7%  
> AMD| 17137| 2.5%  
> NXP Semiconductors| 16716| 2.4%  
> SUSE| 14546| 2.1%  
> Renesas Electronics| 14103| 2.0%  
> IBM| 13068| 1.9%  
> Atomide| 12237| 1.8%  
> Huawei Technologies| 10952| 1.6%  
> Code Aurora Forum| 10566| 1.5%  
> (Consultant)| 9353| 1.4%  
> ARM| 7034| 1.0%  
  
The kernel development community relies heavily on its testers and reviewers. The testing and review picture for 5.0 looks like this: 

> Test and review credits in 5.0   
> ---  
> | Tested-by  
> ---  
> Andrew Bowers| 50| 6.4%  
> Ming Lei| 37| 4.7%  
> Jarkko Sakkinen| 20| 2.6%  
> Arnaldo Carvalho de Melo| 18| 2.3%  
> Janusz Krzysztofik| 17| 2.2%  
> Alan Tull| 17| 2.2%  
> Tony Luck| 16| 2.1%  
> Aaron Brown| 15| 1.9%  
> Jesper Dangaard Brouer| 15| 1.9%  
> Heiko Stuebner| 14| 1.8%  
> Marek Szyprowski| 13| 1.7%  
> Corentin Labbe| 13| 1.7%  
> Adam Ford| 12| 1.5%  
> Wolfram Sang| 11| 1.4%  
> Tom Zanussi| 11| 1.4%  
> Steve Longerbeam| 11| 1.4%  
> Ravulapati Vishnu vardhan Rao| 11| 1.4%  
> David Ahern| 10| 1.3%  
> Jarkko Nikula| 10| 1.3%  
> Ondrej Jirman| 10| 1.3%  
> | Reviewed-by  
> ---  
> Rob Herring| 186| 3.8%  
> Ville Syrjälä| 125| 2.5%  
> Simon Horman| 108| 2.2%  
> Geert Uytterhoeven| 92| 1.9%  
> Hannes Reinecke| 90| 1.8%  
> Christoph Hellwig| 83| 1.7%  
> Alex Deucher| 72| 1.5%  
> David Sterba| 69| 1.4%  
> Andrew Morton| 60| 1.2%  
> Omar Sandoval| 60| 1.2%  
> John Hurley| 58| 1.2%  
> Rodrigo Vivi| 57| 1.2%  
> Chris Wilson| 57| 1.2%  
> Sagi Grimberg| 56| 1.1%  
> Petr Machata| 56| 1.1%  
> Daniel Vetter| 52| 1.1%  
> Christian König| 51| 1.0%  
> Chao Yu| 48| 1.0%  
> Andy Shevchenko| 44| 0.9%  
> Nikolay Borisov| 41| 0.8%  
  
The kernel's repository can tell us who the patches came from, but it is silent on the question of _where_ they came from. Some insights, though, can be had by looking at the time zone stored in the commit time for each patch. For 5.0, the result looks like this: 

> Originating time zone for 5.0 patches  
> ---  
> Offset| Changesets| | Notes  
> -8:00 | 1,676 | ![](https://static.lwn.net/images/redsquare.png) | US west coast  
> -7:00 | 622 | ![](https://static.lwn.net/images/redsquare.png) | US mountain  
> -6:00 | 361 | ![](https://static.lwn.net/images/redsquare.png) | US central  
> -5:00 | 939 | ![](https://static.lwn.net/images/redsquare.png) | US east coast  
> -4:00 | 295 | ![](https://static.lwn.net/images/redsquare.png) |   
> -3:00 | 158 | ![](https://static.lwn.net/images/redsquare.png) | Brazil  
> -2:00 | 105 | ![](https://static.lwn.net/images/redsquare.png) |   
> 0:00 | 1,611 | ![](https://static.lwn.net/images/redsquare.png) | UK  
> +1:00 | 2,812 | ![](https://static.lwn.net/images/redsquare.png) | Western Europe  
> +2:00 | 1,457 | ![](https://static.lwn.net/images/redsquare.png) | Eastern Europe  
> +3:00 | 447 | ![](https://static.lwn.net/images/redsquare.png) | Finland, Russia  
> +5:30 | 513 | ![](https://static.lwn.net/images/redsquare.png) | India  
> +8:00 | 952 | ![](https://static.lwn.net/images/redsquare.png) | China  
> +9:00 | 302 | ![](https://static.lwn.net/images/redsquare.png) | Japan, Korea  
> +10:00 | 99 | ![](https://static.lwn.net/images/redsquare.png) | Australia  
> +11:00 | 140 | ![](https://static.lwn.net/images/redsquare.png) | Australia  
  
A few time zones with less than ten changesets have been omitted from the above table. The association of time zones with countries is, of course, approximate. Daylight savings time can throw things off, as can developers whose systems are not set to their local time. If nothing else, the number of patches with times in UTC is probably higher than the number that actually came from countries in that time zone. There are still a few conclusions that can be drawn, though: it seems clear that an awful lot of kernel work still happens at or just east of the Prime Meridian, for example. 

More than anything else, though, this table highlights something we already knew: the Linux kernel community is truly global in scope. Patches come in at a high rate from all over the world and are integrated in a (usually) smooth manner. In this sense, the 5.0 kernel is just like the many that came before it; it's business as usual in the kernel community.
