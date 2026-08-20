---
title: Statistics from the 4.7 development cycle
url: https://lwn.net/Articles/695993/
date: "August 2, 2016"
category: "Releases-4.7"
author: "By Jonathan Corbet August 2, 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 2, 2016

The 4.7 kernel was [released](<https://lwn.net/Articles/695268/>) on July 24, so longtime readers might be wondering where the usual development statistics are. We're running a little late this time around, but for good reason — Greg Kroah-Hartman obtained information from a large number of developers on who they work for, and we're now able to use that information to produce better numbers. Of course, the overall story hasn't changed a whole lot — kernel development is relatively boring and predictable these days — but each cycle still has a few noteworthy points. 

The 4.7 development cycle saw the merging of 12,283 changesets from 1,582 developers; 232 of those developers appeared in the kernel changelog for the first time. Those changes added just under 300,000 lines to the kernel source and 740 new files to the kernel tree. Of those developers, the most active were: 

> Most active 4.7 developers  
> ---  
> | By changesets  
> ---  
> H Hartley Sweeten| 208| 1.7%  
> Boris Brezillon| 132| 1.1%  
> Al Viro| 127| 1.0%  
> Linus Walleij| 121| 1.0%  
> Geert Uytterhoeven| 120| 1.0%  
> Arnaldo Carvalho de Melo| 110| 0.9%  
> Ville Syrjälä| 105| 0.9%  
> Laxman Dewangan| 101| 0.8%  
> Arnd Bergmann| 97| 0.8%  
> Jes Sorensen| 97| 0.8%  
> Eric Dumazet| 91| 0.7%  
> Dan Carpenter| 88| 0.7%  
> Aneesh Kumar K.V| 79| 0.6%  
> Michal Hocko| 74| 0.6%  
> Chris Wilson| 71| 0.6%  
> Wolfram Sang| 68| 0.6%  
> Florian Westphal| 66| 0.5%  
> James Hogan| 66| 0.5%  
> Daniel Vetter| 64| 0.5%  
> Imre Deak| 62| 0.5%  
> | By changed lines  
> ---  
> Alex Deucher| 37185| 6.4%  
> Rex Zhu| 19912| 3.4%  
> Paul E. McKenney| 14004| 2.4%  
> Thierry Reding| 9170| 1.6%  
> Jinshan Xiong| 8828| 1.5%  
> Yuval Mintz| 8419| 1.4%  
> Jes Sorensen| 6982| 1.2%  
> Chanwoo Choi| 5742| 1.0%  
> H Hartley Sweeten| 5705| 1.0%  
> Varun Prakash| 5703| 1.0%  
> Boris Brezillon| 5347| 0.9%  
> Aneesh Kumar K.V| 5230| 0.9%  
> Tom Zanussi| 5116| 0.9%  
> CK Hu| 5072| 0.9%  
> Ilya Dryomov| 4764| 0.8%  
> Linus Walleij| 4738| 0.8%  
> Maxime Ripard| 4631| 0.8%  
> Mathieu Poirier| 4559| 0.8%  
> Christoph Hellwig| 4232| 0.7%  
> Finn Thain| 4024| 0.7%  
  
By this point it should come as no surprise that H Hartley Sweeten made it to the top of the "by changesets" list with continued work on the Comedi drivers in the staging tree; nearly 8,400 patches have gone into that subsystem since it was merged. Boris Brezillon's work was mostly focused on the memory-technology devices subsystem (and NAND controllers in particular), Al Viro made a number of fundamental changes (including [parallel lookups](<https://lwn.net/Articles/692546/>)) to the virtual filesystem layer and followed the implications of those changes through many filesystems, Linus Walleij has been reworking the GPIO subsystem, and Geert Uytterhoeven worked all over the tree, with an emphasis on various ARM-related subsystems. 

In the "lines changed" column, Alex Deucher continues to work on the massive amdgpu graphics driver; Rex Zhu is also working primarily on that driver. Paul McKenney works with the read-copy-update subsystem, of course; the elevated line count this time around results from some large documentation changes. Thierry Reding works with the NVIDIA Tegra ARM subarchitecture, and Jinshan Xiong made some extensive changes to the Lustre filesystem in the staging tree. 

Often work in the staging tree tends to overshadow everything else when it comes to these lists, but, this time around, only two developers who appear in the top ten on either side were working on staging code. 

There were 222 companies (that we know about) that supported work merged in the 4.7 development cycle — a fairly average figure for recent years. The most active companies this time around were: 

> Most active 4.7 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1786| 14.5%  
> (None)| 968| 7.9%  
> Red Hat| 967| 7.9%  
> (Unknown)| 861| 7.0%  
> Linaro| 633| 5.2%  
> SUSE| 470| 3.8%  
> IBM| 378| 3.1%  
> AMD| 302| 2.5%  
> Samsung| 276| 2.2%  
> Google| 244| 2.0%  
> Renesas Electronics| 244| 2.0%  
> NVIDIA| 231| 1.9%  
> Mellanox| 227| 1.8%  
> Free Electrons| 222| 1.8%  
> ARM| 217| 1.8%  
> Vision Engraving Systems| 208| 1.7%  
> Oracle| 200| 1.6%  
> Imagination Technologies| 193| 1.6%  
> Texas Instruments| 185| 1.5%  
> Broadcom| 141| 1.1%  
> | By lines changed  
> ---  
> Intel| 86056| 14.8%  
> AMD| 69065| 11.8%  
> (None)| 35035| 6.0%  
> Red Hat| 33887| 5.8%  
> IBM| 28102| 4.8%  
> Linaro| 23396| 4.0%  
> (Unknown)| 23287| 4.0%  
> NVIDIA| 18023| 3.1%  
> Mellanox| 14011| 2.4%  
> Samsung| 12918| 2.2%  
> SUSE| 12810| 2.2%  
> Free Electrons| 12637| 2.2%  
> QLogic| 11731| 2.0%  
> ARM| 9000| 1.5%  
> Rockchip| 8938| 1.5%  
> Renesas Electronics| 8734| 1.5%  
> Texas Instruments| 7462| 1.3%  
> (Consultant)| 6964| 1.2%  
> Chelsio| 6868| 1.2%  
> Broadcom| 6564| 1.1%  
  
This table looks as it has for some time, no real surprises here. The percentage of changes from developers working on their own time, at 7.9%, is up from 4.6, but still remains low by historical standards. Once upon a time, volunteer developers were our primary source of new contributors to the kernel. In 4.7, of the 232 first-time contributors, 132 were known to be employed at the time, 38 were known to be working on their own time, and 62 are in the "unknown" column. Even if all the unknowns are volunteers (most of them probably are), we still have more new contributors arriving via companies. 

Contributing to the kernel used to be a fairly reliable way to get a job, and it probably still is. But, in 2016, it seems that many of our new developers get the job first, and it is the job that brings them to the kernel community. 

The table above shows the changes contributed by the most active companies. One last question one might ask is: how many developers does each company have working on Linux? For the 4.7 development cycle, the answer looks like this: 

> # of developers/company  
> ---  
> Company| Count| Percent  
> (Unknown)| 238| 14.5%  
> Intel| 198| 12.1%  
> (None)| 172| 10.5%  
> Red Hat| 91| 5.6%  
> IBM| 64| 3.9%  
> Google| 48| 2.9%  
> Linaro| 43| 2.6%  
> Mellanox| 38| 2.3%  
> SUSE| 37| 2.3%  
> AMD| 30| 1.8%  
> Samsung| 27| 1.6%  
> Huawei Technologies| 27| 1.6%  
> ARM| 25| 1.5%  
> Texas Instruments| 23| 1.4%  
> Broadcom| 22| 1.3%  
> Oracle| 21| 1.3%  
> NXP| 20| 1.2%  
> Qualcomm| 17| 1.0%  
> MediaTek| 13| 0.8%  
> Imagination Technologies| 12| 0.7%  
> Renesas Electronics| 12| 0.7%  
> Facebook| 11| 0.7%  
> NVIDIA| 11| 0.7%  
> Code Aurora Forum| 10| 0.6%  
> (Consultant)| 10| 0.6%  
> Rockchip| 10| 0.6%  
> Canonical| 10| 0.6%  
> Free Electrons| 9| 0.5%  
> Pengutronix| 9| 0.5%  
> Synopsys| 8| 0.5%  
  
Intel, it seems, has far more developers working on the kernel than any other company — nearly 12% of the total in 4.7. Volunteer developers may not contribute a lot of code, but there are quite a few of them; given that many (if not most) of the unknown developers probably fall into this category, developers working on their own time are still the biggest group. 

The kernel community as a whole is a big group indeed, and it continues to produce kernels in a disciplined and predictable way. The relative lack of surprises may make for relatively boring statistics articles, but it is certainly welcome to users of the kernel.
