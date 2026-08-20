---
title: Statistics from the 5.2 kernel — and before
url: https://lwn.net/Articles/791606/
date: "June 21, 2019"
category: "Releases-5.2"
author: "By Jonathan Corbet June 21, 2019"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 21, 2019

As of this writing, just over 13,600 non-merge changesets have been pulled into the mainline repository for the 5.2 development cycle. The time has come, once again, for a look at where that work came from and who supported it. There are some unique aspects to 5.2 that have thrown off some of the usual numbers. 

1,716 developers contributed changes for the 5.2 kernel, 245 of whom made their first contribution during this cycle. Those 1,716 developers removed nearly 490,000 lines of code, which is a lot, but the addition of 596,000 new lines of code means that the kernel still grew by 106,000 lines. The most active developers this time around were: 

> Most active 5.2 developers  
> ---  
> | By changesets  
> ---  
> Thomas Gleixner| 441| 3.2%  
> Alexandre Belloni| 186| 1.4%  
> Yue Haibing| 171| 1.3%  
> Chris Wilson| 168| 1.2%  
> Guenter Roeck| 160| 1.2%  
> Ville Syrjälä| 135| 1.0%  
> Christoph Hellwig| 125| 0.9%  
> Axel Lin| 110| 0.8%  
> David Sterba| 107| 0.8%  
> Christophe Leroy| 106| 0.8%  
> Gustavo A. R. Silva| 104| 0.8%  
> Colin Ian King| 101| 0.7%  
> Bart Van Assche| 99| 0.7%  
> Alex Shi| 96| 0.7%  
> Masahiro Yamada| 93| 0.7%  
> David Ahern| 88| 0.6%  
> Maxime Ripard| 88| 0.6%  
> Andy Shevchenko| 83| 0.6%  
> Arnd Bergmann| 79| 0.6%  
> Laurent Pinchart| 78| 0.6%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 132179| 14.4%  
> Thomas Gleixner| 126547| 13.8%  
> Yan-Hsuan Chuang| 47518| 5.2%  
> Gabriel Krisman Bertazi| 20170| 2.2%  
> Liam Girdwood| 17379| 1.9%  
> Olaf Weber| 13441| 1.5%  
> Andi Kleen| 12232| 1.3%  
> Hans Verkuil| 11709| 1.3%  
> Chris Wilson| 10025| 1.1%  
> Mauro Carvalho Chehab| 9809| 1.1%  
> Vladimir Oltean| 7074| 0.8%  
> Gal Pressman| 6143| 0.7%  
> David Howells| 5652| 0.6%  
> Neil Brown| 5345| 0.6%  
> Linus Walleij| 5023| 0.5%  
> Rob Herring| 5023| 0.5%  
> Tzvetomir Stoyanov| 4998| 0.5%  
> Ryder Lee| 4890| 0.5%  
> Tony Lindgren| 4868| 0.5%  
> Neil Armstrong| 4783| 0.5%  
  
Of the 441 patches that put Thomas Gleixner at the top of the "by changesets" column, 349 were a part of the ongoing effort to [add SPDX tags](<https://lwn.net/Articles/739183/>) to every kernel source file. That still leaves nearly 100 changes for little issues like the the [microarchitectural data sampling vulnerabilities](<https://www.kernel.org/doc/html/latest/x86/mds.html#mds>), reworking the handling of stack traces, and more. Alexandre Belloni contributed a long list of realtime clock changes, Yue Haibing made code cleanups all over the tree, Chris Wilson made many changes to the i915 graphics driver, and Guenter Roeck made a lot of improvements to the hardware monitoring and watchdog subsystems. 

In the "lines changed" column, Greg Kroah-Hartman deleted the rtlwifi driver from the staging tree; it has been superseded by a non-staging driver. One might not think that adding a single SPDX line to files would lead to a lot of changed lines, even when a lot of files are involved, but most of those additions also allowed the removal of a lot of license boilerplate. As a result, Gleixner's SPDX work resulted in the removal of over 100,000 lines from the kernel. Yan-Hsuan Chuang only contributed three changes, but they brought in the new production Realtek driver, which is not small. Gabriel Krisman Bertazi added support for [case-insensitive file-name lookups](<https://lwn.net/Articles/784041/>) to the ext4 filesystem; that, in turn, brought in a large body of automatically-generated code for [UTF-8 handling](<https://lwn.net/Articles/784124/>). Liam Girdwood add the [Sound Open Firmware](<https://lwn.net/Articles/749888/>) system. 

The top testers and reviewers this time around are a bit different from the usual crowd: 

> Test and review credits in 5.2   
> ---  
> | Tested-by  
> ---  
> Andrew Bowers| 132| 16.6%  
> Sebastian Reichel| 45| 5.7%  
> Zhang Lei| 24| 3.0%  
> Leo Yan| 20| 2.5%  
> Robert Walker| 20| 2.5%  
> Arnaldo Carvalho de Melo| 14| 1.8%  
> Jon Hunter| 13| 1.6%  
> Jon Masters| 12| 1.5%  
> Jerome Brunet| 11| 1.4%  
> Brice Goglin| 10| 1.3%  
> Stefan Wahren| 10| 1.3%  
> Aaron Brown| 10| 1.3%  
> Mathieu Malaterre| 9| 1.1%  
> Holger Hoffstätte| 9| 1.1%  
> Bart Van Assche| 8| 1.0%  
> Jeffrey Hugo| 8| 1.0%  
> Oleksandr Natalenko| 8| 1.0%  
> Sudip Mukherjee| 8| 1.0%  
> James Smart| 7| 0.9%  
> Michał Mirosław| 7| 0.9%  
> Robert Yang| 7| 0.9%  
> | Reviewed-by  
> ---  
> Allison Randal| 345| 5.4%  
> Rob Herring| 199| 3.1%  
> Kate Stewart| 174| 2.7%  
> Richard Fontana| 157| 2.4%  
> Alexios Zavras| 120| 1.9%  
> Mukesh Ojha| 116| 1.8%  
> Huang Rui| 107| 1.7%  
> David Sterba| 103| 1.6%  
> Alex Deucher| 100| 1.6%  
> Tvrtko Ursulin| 99| 1.5%  
> Armijn Hemel| 96| 1.5%  
> Florian Fainelli| 85| 1.3%  
> Simon Horman| 79| 1.2%  
> Andrew Lunn| 77| 1.2%  
> Takashi Iwai| 74| 1.2%  
> Chris Wilson| 73| 1.1%  
> Steve Winslow| 65| 1.0%  
> Guenter Roeck| 63| 1.0%  
> Christoph Hellwig| 62| 1.0%  
> Mauro Carvalho Chehab| 60| 0.9%  
  
Andrew Bowers apparently tested a large number of i40e network-driver patches written by co-workers at Intel. Then, there are some familiar names in the review column. People like Allison Randal, Kate Stewart, and Richard Fontana are all quite well known in our community, but they tend not to show up often in kernel patches. In this case, all of them spent time (along with Alexios Zavras and Armijn Hemel) reviewing the SPDX changes. This is important work: the addition of SPDX lines must reflect the actual license of each file and not inadvertently change that license. 

Work on 5.2 was supported by 215 employers (that we know of), the most active of which were: 

> Most active 5.2 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1684| 12.4%  
> (Unknown)| 1063| 7.8%  
> Red Hat| 859| 6.3%  
> Google| 730| 5.4%  
> (None)| 633| 4.7%  
> AMD| 544| 4.0%  
> Linutronix| 480| 3.5%  
> SUSE| 423| 3.1%  
> IBM| 411| 3.0%  
> Linaro| 370| 2.7%  
> Huawei Technologies| 367| 2.7%  
> Bootlin| 360| 2.6%  
> Mellanox| 354| 2.6%  
> ARM| 305| 2.2%  
> (Consultant)| 295| 2.2%  
> Renesas Electronics| 276| 2.0%  
> Oracle| 217| 1.6%  
> NXP Semiconductors| 182| 1.3%  
> Linux Foundation| 160| 1.2%  
> BayLibre| 159| 1.2%  
> | By lines changed  
> ---  
> Linux Foundation| 132863| 14.5%  
> Linutronix| 127352| 13.9%  
> Intel| 99721| 10.9%  
> Realtek| 47705| 5.2%  
> (Unknown)| 35515| 3.9%  
> Red Hat| 34869| 3.8%  
> AMD| 23993| 2.6%  
> Collabora Multimedia| 22794| 2.5%  
> Linaro| 22696| 2.5%  
> Google| 22450| 2.5%  
> (None)| 22200| 2.4%  
> IBM| 19651| 2.1%  
> MediaTek| 14937| 1.6%  
> Mellanox| 14657| 1.6%  
> Cisco| 14656| 1.6%  
> Samsung| 14260| 1.6%  
> SUSE| 13559| 1.5%  
> SGI| 13441| 1.5%  
> ARM| 12407| 1.4%  
> BayLibre| 11753| 1.3%  
  
There are few surprises here, as usual. It is nice to see Realtek showing up as a top contributor, though — a change from the recent past. 

When developers apply a Signed-off-by tag to a patch that they did not write, it is usually an indication that they are merging that patch into a subsystem tree. A look at non-author signoffs thus gives an idea of who the gatekeepers for the kernel are. In 5.2, the most active developers in this role, and the companies that supported them, are: 

> Most non-author signoffs  
> ---  
> | Individuals  
> ---  
> David S. Miller| 1214| 9.4%  
> Greg Kroah-Hartman| 1191| 9.2%  
> Mark Brown| 643| 5.0%  
> Alex Deucher| 530| 4.1%  
> Martin K. Petersen| 318| 2.5%  
> Andrew Morton| 304| 2.4%  
> Mauro Carvalho Chehab| 222| 1.7%  
> Ingo Molnar| 209| 1.6%  
> Michael Ellerman| 208| 1.6%  
> Jason Gunthorpe| 194| 1.5%  
> Jens Axboe| 187| 1.5%  
> Jonathan Cameron| 177| 1.4%  
> Jonathan Corbet| 173| 1.3%  
> Herbert Xu| 165| 1.3%  
> Alexei Starovoitov| 149| 1.2%  
> Jeff Kirsher| 146| 1.1%  
> David Sterba| 143| 1.1%  
> Hans Verkuil| 140| 1.1%  
> Shawn Guo| 138| 1.1%  
> Daniel Borkmann| 126| 1.0%  
> | Employers  
> ---  
> Red Hat| 2256| 17.5%  
> Linux Foundation| 1255| 9.7%  
> Linaro| 1231| 9.6%  
> Intel| 1045| 8.1%  
> Google| 704| 5.5%  
> AMD| 556| 4.3%  
> (None)| 417| 3.2%  
> Mellanox| 397| 3.1%  
> IBM| 394| 3.1%  
> Facebook| 394| 3.1%  
> SUSE| 382| 3.0%  
> Oracle| 357| 2.8%  
> ARM| 333| 2.6%  
> Huawei Technologies| 330| 2.6%  
> Samsung| 322| 2.5%  
> (Unknown)| 246| 1.9%  
> Bootlin| 198| 1.5%  
> LWN.net| 173| 1.3%  
> Code Aurora Forum| 146| 1.1%  
> (Consultant)| 141| 1.1%  
  
The concentration of subsystem maintainers into a small number of companies has spread out over time, but it is still true that (just) over 50% of the non-author signoffs are made by developers working in five companies. 

#### Code longevity

Finally, it has been a while since we have looked at how much code from each development cycle remains in the 5.2 kernel. This determination is made through the application of a fair amount of brute force: for each file in the kernel tree, `git blame` is run and the commit that added each line is noted. After associating each commit with the development cycle that brought it into the mainline, it is possible to see how many lines of code were introduced by each cycle. The results look like this: 

> ![\[Bar chart\]](https://static.lwn.net/images/2019/5.2-lines.svg)

As can be seen here, newer releases are more heavily represented than older ones, which is not surprising. There has been more time for changes to be made to code from older releases, and more recent cycles tend to add more lines in general. Still, it seems clear that a lot of the code we add stays in the kernel for a long time. 

The most graphic illustration of that, actually, does not appear in the graph because it would squeeze the rest into insignificance: over 2.5 million lines of code in the kernel — nearly 10% of the total — date back to the initial Git commit in 2005. Much of that code was old even then, and certainly hasn't gotten any younger. 

Stepping back one release, there are [215 files in 5.1](<https://lwn.net/Articles/791608/>) that have not seen a single change since the initial commit. This number can only undercount the number of dormant files by a significant factor: any file that has seen even a single whitespace fix, coding-style change, or typo fix will not appear in this list. Indeed, the list was generated from the 5.1 kernel because the SPDX work in 5.2 is obscuring things further. These files may get shiny new license tags, but that won't make them any more alive. 

Code that is this old perhaps just reached perfection years ago and needs no further changes. In general, though, static code is often unmaintained, unused, and possibly insecure. For anybody looking to do a little cleanup work, this list might not be a bad place to start.
