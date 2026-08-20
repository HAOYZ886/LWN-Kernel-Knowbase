---
title: Statistics from the 5.4 development cycle
url: https://lwn.net/Articles/804119/
date: "November 7, 2019"
category: "Releases-5.4"
author: "By Jonathan Corbet November 7, 2019"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 7, 2019

As of this writing, just over 14,000 non-merge changesets have found their way into the mainline repository for the 5.4 release; that is a bit less than we saw for 5.3, but more than most of the other recent kernels. The final 5.4 release is approaching, so it must be time for our usual look at where the code merged in this development cycle came from. It's mostly business as usual in the kernel community, modulo an appearance from none other than Hulk Robot. 

Those 14,000 changesets were contributed by 1,802 developers, which is just short of the 1,846 who contributed to 5.3; there is still time, though, for 5.4 to set a new record for the number of contributors — a surprising number of developers wait until the end of the release cycle to fix something. Of the developers seen so far, 266 made their first contribution to the kernel in this cycle. The combined work from these developers increased the size of the kernel by 393,000 lines. 

The most active developers this time around were: 

> Most active 5.4 developers  
> ---  
> | By changesets  
> ---  
> Yue Haibing| 270| 1.9%  
> Chris Wilson| 238| 1.7%  
> Arnaldo Carvalho de Melo| 182| 1.3%  
> Christoph Hellwig| 165| 1.2%  
> Jiri Olsa| 165| 1.2%  
> Nishka Dasgupta| 150| 1.1%  
> Andy Shevchenko| 145| 1.0%  
> Colin Ian King| 140| 1.0%  
> Masahiro Yamada| 121| 0.9%  
> Ard Biesheuvel| 116| 0.8%  
> Kuninori Morimoto| 103| 0.7%  
> Andrey Smirnov| 99| 0.7%  
> Thomas Gleixner| 98| 0.7%  
> Greg Kroah-Hartman| 97| 0.7%  
> Linus Walleij| 92| 0.7%  
> Xiaojie Yuan| 91| 0.6%  
> Chuhong Yuan| 89| 0.6%  
> Arnd Bergmann| 89| 0.6%  
> Daniel Vetter| 83| 0.6%  
> Maxime Ripard| 80| 0.6%  
> | By changed lines  
> ---  
> Le Ma| 92046| 10.1%  
> Bhawanpreet Lakha| 89403| 9.8%  
> Christoph Hellwig| 45376| 5.0%  
> Arnd Bergmann| 35746| 3.9%  
> Ard Biesheuvel| 17750| 2.0%  
> Chris Wilson| 12697| 1.4%  
> Valdis Klētnieks| 10941| 1.2%  
> Alex Vesker| 10937| 1.2%  
> Shannon Nelson| 9740| 1.1%  
> Eric Biggers| 8120| 0.9%  
> Pawel Laszczak| 7824| 0.9%  
> Neil Armstrong| 7370| 0.8%  
> Jun Lei| 7338| 0.8%  
> Jiri Olsa| 7334| 0.8%  
> Yan-Hsuan Chuang| 7087| 0.8%  
> Andrew Jeffery| 6827| 0.8%  
> Paul Cercueil| 6217| 0.7%  
> Leo Liu| 6122| 0.7%  
> Larry Finger| 6077| 0.7%  
> Mauro Carvalho Chehab| 5945| 0.7%  
  
At the top of the "by changesets" column is Yue Haibing, who fixed a large number of small problems all over the kernel. The bulk of these problems, it seems, have been found using a system at Huawei called Hulk Robot, which uses various Coccinelle scripts and reports the results. Chris Wilson continues to contribute a lot of work to the Intel i915 graphics driver. Arnaldo Carvalho de Melo worked almost exclusively on the perf tool, Christoph Hellwig worked throughout the memory-management and I/O subsystems (and also removed a lot of obsolete code), and Jiri Olsa worked on perf as well. 

Le Ma added 92,000 lines of code which was, as is often the case for the weightiest patches, mostly register definitions for the amdgpu graphics driver; Bhawanpreet Lakha also worked on the AMD graphics subsystem. Arnd Bergmann, in addition to fixing problems across the kernel tree, removed a number of obsolete drivers, and Ard Biesheuvel worked almost exclusively within the crypto subsystem. 

Work on 5.4 was supported by 234 employers that we were able to identify; the most active of those were: 

> Most active 5.4 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1714| 12.2%  
> Red Hat| 1048| 7.4%  
> (Unknown)| 931| 6.6%  
> AMD| 859| 6.1%  
> (None)| 818| 5.8%  
> Google| 759| 5.4%  
> Huawei Technologies| 650| 4.6%  
> IBM| 556| 3.9%  
> Linaro| 546| 3.9%  
> Mellanox| 357| 2.5%  
> NXP Semiconductors| 334| 2.4%  
> Renesas Electronics| 307| 2.2%  
> SUSE| 280| 2.0%  
> (Consultant)| 273| 1.9%  
> ARM| 225| 1.6%  
> Linux Foundation| 182| 1.3%  
> BayLibre| 174| 1.2%  
> Canonical| 173| 1.2%  
> Broadcom| 162| 1.2%  
> Oracle| 161| 1.1%  
> | By lines changed  
> ---  
> AMD| 229309| 25.3%  
> Intel| 75357| 8.3%  
> Linaro| 66064| 7.3%  
> (Consultant)| 51674| 5.7%  
> Red Hat| 39670| 4.4%  
> IBM| 33541| 3.7%  
> Google| 33363| 3.7%  
> (None)| 33362| 3.7%  
> (Unknown)| 32469| 3.6%  
> Mellanox| 27268| 3.0%  
> NXP Semiconductors| 12879| 1.4%  
> Realtek| 11655| 1.3%  
> BayLibre| 11447| 1.3%  
> Cisco| 11378| 1.3%  
> Facebook| 11010| 1.2%  
> Virginia Tech| 10941| 1.2%  
> Huawei Technologies| 10291| 1.1%  
> Pensando| 9740| 1.1%  
> SUSE| 8887| 1.0%  
> Code Aurora Forum| 8716| 1.0%  
  
As usual, there is not much in the way of surprises here. The 650 changesets from Huawei are a new high for that company; Huawei may be having significant political problems in the market, but its contributions at the code level continue unabated. 

The testing and reviewing numbers this time around look like this: 

> Test and review credits in 5.4   
> ---  
> | Tested-by  
> ---  
> Andrew Bowers| 162| 16.9%  
> Himanshu Madhani| 60| 6.3%  
> Arnaldo Carvalho de Melo| 33| 3.4%  
> Artur Rojek| 25| 2.6%  
> Kevin Hilman| 18| 1.9%  
> Ralph Campbell| 18| 1.9%  
> Aaron Brown| 16| 1.7%  
> Geert Uytterhoeven| 13| 1.4%  
> Sedat Dilek| 12| 1.3%  
> Mathieu Malaterre| 12| 1.3%  
> | Reviewed-by  
> ---  
> Chris Wilson| 219| 3.5%  
> Rob Herring| 201| 3.2%  
> Christoph Hellwig| 143| 2.3%  
> Alex Deucher| 140| 2.3%  
> Hawking Zhang| 115| 1.8%  
> Sam Ravnborg| 88| 1.4%  
> David Sterba| 87| 1.4%  
> Andrew Lunn| 86| 1.4%  
> Emil Velikov| 85| 1.4%  
> Simon Horman| 71| 1.1%  
  
Andrew Bowers still has never contributed a patch to the kernel, but he collects Tested-by credits for patches from many other Intel developers for patches to the company's network drivers. The patches tested vary from significant changes to [version-number tweaks](<https://git.kernel.org/linus/2de125663658>). The top reviewer, Chris Wilson, is also based at Intel; when he is not writing i915 patches he is busily reviewing i915 patches written by others. Those patches are mostly, but not exclusively, from Intel developers as well. 

The situation with Reported-by credits, given to those who report bugs, looks like this: 

> Reported-by credits in 5.4   
> ---  
> | Recipients  
> ---  
> Hulk Robot| 263| 25.4%  
> Syzbot| 85| 8.2%  
> kbuild test robot| 71| 6.8%  
> Dan Carpenter| 28| 2.7%  
> Stephen Rothwell| 26| 2.5%  
> Christoph Hellwig| 25| 2.4%  
> Qian Cai| 10| 1.0%  
> Geert Uytterhoeven| 9| 0.9%  
> Andrew Lunn| 8| 0.8%  
> Arnd Bergmann| 6| 0.6%  
> Russell King| 6| 0.6%  
> Randy Dunlap| 6| 0.6%  
> Guenter Roeck| 6| 0.6%  
> Michal Kubecek| 6| 0.6%  
> | Creditors  
> ---  
> Yue Haibing| 236| 22.8%  
> Eric Dumazet| 34| 3.3%  
> Gao Xiang| 30| 2.9%  
> zhengbin| 25| 2.4%  
> Will Deacon| 14| 1.4%  
> Chris Wilson| 14| 1.4%  
> Gustavo A. R. Silva| 14| 1.4%  
> Jens Axboe| 14| 1.4%  
> Nathan Chancellor| 13| 1.3%  
> Stephen Boyd| 10| 1.0%  
  
It is worth noting that, for previous kernels, the number of report credits for Syzbot, the fuzzing tool run at Google, has been significantly underreported. Syzbot requests credit in a unique format that, among other things, uses a different email address for each one; for 5.4, the gitdm tool has been enhanced to recognize and coalesce those credits to yield the correct number. 

What jumps out here, though, is Hulk Robot, which is credited with reporting fully 25% of the bugs fixed in this development cycle. In the other column, we see Yue Haibing as the developer who added the most Reported-by tags to patches. Those two numbers are not unrelated: Haibing is the developer using this system to find bugs to fix. It is arguably not normal practice for developers to give report credits to bugs they find and fix themselves, but it does shine a light on how one developer is fixing large numbers of problems. 

The increased use of automated testing systems is clearly having an impact; three such systems are now earning 40% of the bug-report credits, and they all far outpace any human reporters. Note that, contrary to some rumors, neither Dan Carpenter nor Stephen Rothwell is a robot. More seriously, there have been worries that Intel's 0day testing service ("kbuild test robot" above) is not proving as useful as it once was, but it still generated a fair number of bug reports during this cycle. 

Overall, the numbers show that the kernel community continues to generate change at a high rate, and to bring in a significant number of new developers with each release. The fixing of problems seems to be happening at a high rate; hopefully they are being fixed more quickly than ones are being added. Either way, it looks like there will be plenty for the development community to do for the foreseeable future.
