---
title: Development statistics for the 5.11 kernel
url: https://lwn.net/Articles/845831/
date: "February 15, 2021"
category: "Releases-5.11"
author: "By Jonathan Corbet February 15, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 15, 2021

The 5.11 kernel was [released](<https://lwn.net/Articles/846222/>) on February 14 — the most romantic sort of Valentine's day gift one could hope for. This kernel saw the merging of 14,340 changesets from 1,912 developers; it is certainly not the busiest development cycle we have seen recently, but it still saw a lot of activity. Read on for our traditional look at where the code merged for 5.11 came from. 

The history of the 5.x kernels to date looks like this: 

> Release| Changesets| Developers  
> ---|---|---  
> 5.0 | 12,808 | 1,760  
> 5.1 | 13,034 | 1,727  
> 5.2 | 14,024 | 1,784  
> 5.3 | 14,605 | 1,882  
> 5.4 | 14,619 | 1,877  
> 5.5 | 14,350 | 1,885  
> 5.6 | 12,665 | 1,712  
> 5.7 | 13,901 | 1,878  
> 5.8 | 16,306 | 1,991  
> 5.9 | 14,858 | 1,917  
> 5.10 | 16,174 | 1,971  
> 5.11 | 14,340 | 1,912  
  
The 5.11 development cycle, thus, looks fairly average and unremarkable in general — a middle-of-the road 5.x kernel. That said, the number of developers involved remains over 1,900; 280 of those developers made their first kernel contribution during the 5.11 cycle. The 5.11 kernel contains 608,000 more lines of code than 5.10 did. 

The most active 5.11 developers were: 

> Most active 5.11 developers  
> ---  
> | By changesets  
> ---  
> Lee Jones| 497| 3.5%  
> Krzysztof Kozlowski| 195| 1.4%  
> Thomas Gleixner| 148| 1.0%  
> Christophe Leroy| 135| 0.9%  
> Chuck Lever| 133| 0.9%  
> Christoph Hellwig| 132| 0.9%  
> Sakari Ailus| 126| 0.9%  
> Arnd Bergmann| 119| 0.8%  
> Ville Syrjälä| 119| 0.8%  
> Jonathan Cameron| 115| 0.8%  
> Gustavo A. R. Silva| 93| 0.6%  
> Tom Rix| 90| 0.6%  
> Andy Shevchenko| 89| 0.6%  
> Geert Uytterhoeven| 87| 0.6%  
> Johannes Berg| 86| 0.6%  
> Alex Elder| 83| 0.6%  
> Takashi Iwai| 83| 0.6%  
> Christian König| 82| 0.6%  
> Colin Ian King| 81| 0.6%  
> Trond Myklebust| 81| 0.6%  
> | By changed lines  
> ---  
> Huang Rui| 269436| 28.0%  
> Bhawanpreet Lakha| 81661| 8.5%  
> Lee Jones| 17973| 1.9%  
> Oded Gabbay| 14281| 1.5%  
> Jiaxin Yu| 13408| 1.4%  
> Johannes Berg| 9764| 1.0%  
> Sakari Ailus| 9305| 1.0%  
> Ilya Dryomov| 7528| 0.8%  
> Srinivas Kandagatla| 6072| 0.6%  
> Jonathan Cameron| 5958| 0.6%  
> Fabio Estevam| 5913| 0.6%  
> Christian König| 5789| 0.6%  
> Roman Li| 5370| 0.6%  
> Arnd Bergmann| 5084| 0.5%  
> Chuck Lever| 4681| 0.5%  
> Thomas Gleixner| 4672| 0.5%  
> Christoph Hellwig| 4612| 0.5%  
> Jin Yao| 4577| 0.5%  
> Jarkko Sakkinen| 4575| 0.5%  
> Peter Geis| 4521| 0.5%  
  
Lee Jones was the contributor of the largest number of changesets this time around; the bulk of that work went into fixing compiler warnings throughout the tree. Krzysztof Kozlowski continues to contribute cleanups and small improvements to drivers and devicetree files. Thomas Gleixner worked on a lot of low-level improvements, including the [`kmap_local()`](<https://lwn.net/Articles/836144/>) mechanism. Christophe Leroy did a lot of work in the PowerPC architecture code, and Chuck Lever contributed a large number of (mostly) NFS-server changes. 

The largest code addition — by far — was by Huang Rui, who added yet another set of massive amdgpu header files; Bhawanpreet Lakha also worked on the AMD graphics drivers. Oded Gabbay added support for the Habana "Gaudi" processor, which inevitably included a set of large header files, and Jiaxin Yu added the mt8192 audio driver. 

Work on the 5.11 kernel was supported by 225 companies that we were able to identify; that is a typical number. The most active of those companies were: 

> Most active 5.11 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1364| 9.5%  
> (Unknown)| 1106| 7.7%  
> Linaro| 1050| 7.3%  
> Red Hat| 836| 5.8%  
> AMD| 825| 5.8%  
> Huawei Technologies| 766| 5.3%  
> Google| 698| 4.9%  
> (None)| 523| 3.6%  
> SUSE| 451| 3.1%  
> IBM| 432| 3.0%  
> NVIDIA| 400| 2.8%  
> Facebook| 390| 2.7%  
> Arm| 313| 2.2%  
> Samsung| 288| 2.0%  
> NXP Semiconductors| 284| 2.0%  
> (Consultant)| 255| 1.8%  
> Oracle| 250| 1.7%  
> Linutronix| 233| 1.6%  
> Renesas Electronics| 222| 1.5%  
> Code Aurora Forum| 205| 1.4%  
> | By lines changed  
> ---  
> AMD| 382664| 39.7%  
> Intel| 94102| 9.8%  
> Linaro| 47288| 4.9%  
> (Unknown)| 39747| 4.1%  
> (None)| 28334| 2.9%  
> Google| 25976| 2.7%  
> Red Hat| 24926| 2.6%  
> MediaTek| 19657| 2.0%  
> NVIDIA| 18029| 1.9%  
> Huawei Technologies| 17853| 1.9%  
> NXP Semiconductors| 15108| 1.6%  
> IBM| 14767| 1.5%  
> SUSE| 14337| 1.5%  
> Facebook| 13033| 1.4%  
> (Consultant)| 12716| 1.3%  
> Code Aurora Forum| 12491| 1.3%  
> Arm| 10065| 1.0%  
> BayLibre| 9688| 1.0%  
> Linutronix| 8843| 0.9%  
> Texas Instruments| 7916| 0.8%  
  
As usual, there are few surprises here. If there is any sort of trend to point out in these results, it is that contributions from hardware companies are slowly growing in proportion relative to those from software and support companies. 

#### Drilling down

The above numbers are all relative to the kernel source tree as a whole. If one looks at specific subsystems, the picture changes a bit. The results at this level tend to be more variable from one release to the next, so it makes sense to look over a long time period. The following numbers, thus, are accumulated over the time since the 5.5 release in January 2020; they are, in other words, a first approximation of the work that was merged last year. 

Much of the core kernel code lives in the `kernel` directory. The developers and companies most actively working in that directory during 2020 were: 

> Most active (`kernel`)  
> ---  
> | Developers  
> ---  
> Paul E. McKenney| 250| 9.0%  
> Peter Zijlstra| 140| 5.1%  
> Christoph Hellwig| 110| 4.0%  
> Thomas Gleixner| 105| 3.8%  
> Steven Rostedt| 98| 3.5%  
> Yonghong Song| 59| 2.1%  
> Marco Elver| 58| 2.1%  
> Masami Hiramatsu| 56| 2.0%  
> Eric W. Biederman| 44| 1.6%  
> Andrii Nakryiko| 40| 1.4%  
> | Companies  
> ---  
> Facebook| 486| 17.6%  
> Intel| 294| 10.6%  
> Google| 274| 9.9%  
> Red Hat| 228| 8.2%  
> Linutronix| 161| 5.8%  
> (Unknown)| 141| 5.1%  
> Linaro| 121| 4.4%  
> (Consultant)| 112| 4.0%  
> VMware| 98| 3.5%  
> Arm| 97| 3.5%  
  
Over the last few years, Facebook has become the home to an increasing number of core-kernel developers, to the point that the company contributes far more patches to the `kernel` directory than any other. 

The picture shifts a bit if one looks at patches to the memory-management code, found in the `mm` subdirectory. 

> Most active (`mm`)  
> ---  
> | Developers  
> ---  
> Christoph Hellwig| 82| 5.4%  
> Matthew Wilcox| 65| 4.3%  
> David Hildenbrand| 60| 4.0%  
> Wei Yang| 57| 3.8%  
> Roman Gushchin| 51| 3.4%  
> Andrey Konovalov| 48| 3.2%  
> Johannes Weiner| 45| 3.0%  
> Vlastimil Babka| 36| 2.4%  
> John Hubbard| 34| 2.2%  
> Alex Shi| 30| 2.0%  
> Mike Rapoport| 30| 2.0%  
> | Companies  
> ---  
> Red Hat| 162| 10.7%  
> Google| 156| 10.3%  
> Facebook| 121| 8.0%  
> (Unknown)| 117| 7.7%  
> Oracle| 95| 6.3%  
> (Consultant)| 85| 5.6%  
> (None)| 75| 5.0%  
> SUSE| 73| 4.8%  
> Alibaba| 73| 4.8%  
> Huawei Technologies| 71| 4.7%  
> Intel| 68| 4.5%  
  
The picture for filesystems looks like this: 

> Most active (`fs`)  
> ---  
> | Developers  
> ---  
> Pavel Begunkov| 343| 5.8%  
> Christoph Hellwig| 322| 5.4%  
> Jens Axboe| 264| 4.4%  
> Darrick J. Wong| 244| 4.1%  
> Josef Bacik| 228| 3.8%  
> Trond Myklebust| 207| 3.5%  
> Nikolay Borisov| 199| 3.3%  
> Al Viro| 174| 2.9%  
> Eric Biggers| 154| 2.6%  
> Qu Wenruo| 145| 2.4%  
> | Companies  
> ---  
> Red Hat| 1043| 17.5%  
> SUSE| 788| 13.2%  
> Facebook| 690| 11.6%  
> Oracle| 547| 9.2%  
> Huawei Technologies| 389| 6.5%  
> (Consultant)| 365| 6.1%  
> Google| 347| 5.8%  
> (None)| 314| 5.3%  
> (Unknown)| 285| 4.8%  
> Hammerspace| 207| 3.5%  
  
Much of the "filesystem" work over last year has been focused on the rapidly developing io_uring subsystem and on core infrastructural work. Beyond that, XFS, Btrfs, and NFS were all areas of active development. 

And if one looks at the networking subsystem, the results are: 

> Most active (`net`)  
> ---  
> | Developers  
> ---  
> Chuck Lever| 188| 4.1%  
> Christoph Hellwig| 169| 3.7%  
> Paolo Abeni| 136| 3.0%  
> Florian Westphal| 124| 2.7%  
> Eric Dumazet| 115| 2.5%  
> Karsten Graul| 106| 2.3%  
> Pablo Neira Ayuso| 105| 2.3%  
> Johannes Berg| 100| 2.2%  
> Vladimir Oltean| 83| 1.8%  
> Xin Long| 77| 1.7%  
> | Companies  
> ---  
> Red Hat| 573| 12.5%  
> Google| 430| 9.4%  
> Intel| 383| 8.3%  
> (None)| 308| 6.7%  
> (Unknown)| 303| 6.6%  
> (Consultant)| 238| 5.2%  
> Oracle| 219| 4.8%  
> Mellanox| 183| 4.0%  
> IBM| 179| 3.9%  
> Huawei Technologies| 172| 3.7%  
  
There are a couple of interesting results here. Chuck Lever shows up for his work on the RPC code used by the NFS filesystem, for example, while Christoph Hellwig's work was mostly driven by improvements he was making elsewhere in the tree; neither is normally seen as a core networking developer. Paolo Abeni and Florian Westphal both worked on the mostly under-the-radar upstreaming of the [multipath TCP](<https://lwn.net/Articles/544399/>) code, which has been long in coming. 

One conclusion that can be drawn here is that core-kernel work is somewhat concentrated in a relatively small number of companies. That said, it is much less so than it was some years ago. It would appear that more companies working within Linux have come to the conclusion that their interests lie in working beyond the specific subsystems needed to make their own hardware work. That seems like a good thing for the long-term sustainability of the kernel.
