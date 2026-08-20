---
title: Statistics from the 4.18 development cycle
url: https://lwn.net/Articles/760690/
date: "July 24, 2018"
category: "Releases-4.18"
author: "By Jonathan Corbet July 24, 2018"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 24, 2018

The [4.18-rc6](<https://lwn.net/Articles/760527/>) kernel prepatch came out on July 22, right on schedule. That is a sign that this development cycle is approaching its conclusion, so the time has come for a look at some statistics for how things went this time around. It was another fairly ordinary release cycle for the most part, but with a couple of distinctive features. 

As of 4.18-rc6, 12,879 non-merge changesets had found their way into the mainline repository. This work was contributed by 1,668 developers who added 553,000 lines of code and removed 652,000 lines, for a net reduction of 99,000 lines. This will be the fourth time in the project's history that a release is smaller than its predecessor — and the first time that this has happened for two releases in a row. Of those 1,668 developers, 226 made their first contribution to the kernel this time around; that is the smallest number of first-time contributors since 4.5 was released in March 2016. 

More generally, the number of first-time contributors to each release since 3.0 looks like this: 

> ![\[First-time contributors\]](https://static.lwn.net/images/2018/4.18-first.svg)

While the number of new contributors varies a bit over time, it has remained consistently between 200 and 300 for each development cycle for a long time. New contributors are important to the health of any development project, so it is good that the kernel continues to attract developers over time. 

The most active developers for 4.18 were: 

> Most active 4.18 developers  
> ---  
> | By changesets  
> ---  
> Christoph Hellwig| 218| 1.7%  
> Sergio Paracuellos| 203| 1.6%  
> Ben Skeggs| 162| 1.3%  
> Mauro Carvalho Chehab| 159| 1.2%  
> Colin Ian King| 137| 1.1%  
> Geert Uytterhoeven| 112| 0.9%  
> Chris Wilson| 111| 0.9%  
> Christian Lütke‑Stetzkamp| 109| 0.8%  
> Arnaldo Carvalho de Melo| 108| 0.8%  
> Arnd Bergmann| 106| 0.8%  
> Ajay Singh| 106| 0.8%  
> Fabio Estevam| 94| 0.7%  
> David Ahern| 87| 0.7%  
> Neil Brown| 83| 0.6%  
> Masahiro Yamada| 81| 0.6%  
> Darrick J. Wong| 77| 0.6%  
> Hans de Goede| 75| 0.6%  
> Quytelda Kahja| 75| 0.6%  
> Jakub Kicinski| 69| 0.5%  
> Wolfram Sang| 68| 0.5%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 207274| 20.1%  
> Sakari Ailus| 168085| 16.3%  
> Eric Biggers| 32062| 3.1%  
> Ben Skeggs| 17368| 1.7%  
> Ondrej Mosnacek| 15787| 1.5%  
> Christoph Hellwig| 10553| 1.0%  
> Srinivas Kandagatla| 9984| 1.0%  
> Ian Kent| 7834| 0.8%  
> Alexandre Belloni| 6801| 0.7%  
> Martin KaFai Lau| 6518| 0.6%  
> John Fastabend| 6479| 0.6%  
> Oleksandr Andrushchenko| 6203| 0.6%  
> Steven Eckhoff| 5993| 0.6%  
> Felix Kuehling| 5886| 0.6%  
> Mathieu Desnoyers| 5626| 0.5%  
> Dave Chinner| 5588| 0.5%  
> Kai Chieh Chuang| 5584| 0.5%  
> Manivannan Sadhasivam| 5311| 0.5%  
> Christian Lütke‑Stetzkamp| 5272| 0.5%  
> Niklas Söderlund| 5112| 0.5%  
  
Christoph Hellwig ended up at the top of the per-changesets list with work throughout the block, virtual filesystem, and driver subsystems, including the since-reverted [AIO polling](<https://lwn.net/Articles/743714/>) interface. Sergio Paracuellos made a number of improvements to a couple of staging drivers, Ben Skeggs did a great deal of work on the Nouveau driver as usual, Mauro Carvalho Chehab's work was mostly in the media subsystem (of which he is the maintainer), and Colin Ian King continued his work fixing spelling errors and similar issues throughout the tree. 

In the lines-changed column, Greg Kroah-Hartman [removed the Lustre filesystem](<https://lwn.net/Articles/756565/>) and the ncpfs filesystem as well. Sakari Ailus removed the atomisp driver (which was also in staging), Eric Biggers did a bunch of cryptography-related work (removing a bunch of code in the process), and Ondrej Mosnacek added some new optimized crypto algorithm implementations. 

The developers working on 4.18 were supported by 233 companies that we were able to identify. The most active employers this time around were: 

> Most active 4.18 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1218| 9.5%  
> (None)| 1008| 7.8%  
> Red Hat| 965| 7.5%  
> (Unknown)| 718| 5.6%  
> AMD| 587| 4.6%  
> IBM| 553| 4.3%  
> Linaro| 485| 3.8%  
> Renesas Electronics| 443| 3.4%  
> Google| 380| 3.0%  
> SUSE| 371| 2.9%  
> Samsung| 356| 2.8%  
> (Consultant)| 335| 2.6%  
> Mellanox| 281| 2.2%  
> Huawei Technologies| 266| 2.1%  
> Oracle| 255| 2.0%  
> Facebook| 226| 1.8%  
> Orbital Critical Systems| 203| 1.6%  
> Bootlin| 184| 1.4%  
> Code Aurora Forum| 183| 1.4%  
> Canonical| 176| 1.4%  
> | By lines changed  
> ---  
> Intel| 229121| 22.2%  
> Linux Foundation| 208382| 20.2%  
> Red Hat| 58057| 5.6%  
> Google| 49540| 4.8%  
> AMD| 35006| 3.4%  
> (None)| 31371| 3.0%  
> Linaro| 29845| 2.9%  
> (Unknown)| 26953| 2.6%  
> IBM| 24816| 2.4%  
> Renesas Electronics| 23568| 2.3%  
> Bootlin| 20972| 2.0%  
> Code Aurora Forum| 19634| 1.9%  
> Facebook| 17391| 1.7%  
> Samsung| 17185| 1.7%  
> (Academia)| 16786| 1.6%  
> (Consultant)| 13790| 1.3%  
> Mellanox| 13353| 1.3%  
> MediaTek| 12135| 1.2%  
> SUSE| 10309| 1.0%  
> Oracle| 9105| 0.9%  
  
If a developer applies a Signed-off-by tag to a patch that they are not the author of, it usually means that said developer was the maintainer who applied the patch and set it on the path to mainline inclusion. Looking at non-author signoffs (and associated employers) for 4.18 yields a table that looks like this: 

> Non-author signoffs in 4.18   
> ---  
> | By developer  
> ---  
> David S. Miller| 1304| 10.7%  
> Greg Kroah-Hartman| 1117| 9.2%  
> Alex Deucher| 477| 3.9%  
> Mark Brown| 362| 3.0%  
> Mauro Carvalho Chehab| 346| 2.9%  
> Martin K. Petersen| 291| 2.4%  
> Daniel Borkmann| 261| 2.2%  
> Kalle Valo| 261| 2.2%  
> Michael Ellerman| 235| 1.9%  
> Simon Horman| 183| 1.5%  
> Andrew Morton| 173| 1.4%  
> Jens Axboe| 171| 1.4%  
> Jonathan Cameron| 169| 1.4%  
> Ingo Molnar| 162| 1.3%  
> David Sterba| 159| 1.3%  
> Rafael J. Wysocki| 141| 1.2%  
> Thomas Gleixner| 139| 1.1%  
> Alexei Starovoitov| 127| 1.0%  
> Linus Walleij| 125| 1.0%  
> Hans Verkuil| 121| 1.0%  
> | By employer  
> ---  
> Red Hat| 2242| 18.5%  
> Linux Foundation| 1135| 9.4%  
> Linaro| 959| 7.9%  
> Intel| 928| 7.6%  
> AMD| 572| 4.7%  
> Samsung| 489| 4.0%  
> Google| 441| 3.6%  
> IBM| 439| 3.6%  
> SUSE| 402| 3.3%  
> Oracle| 391| 3.2%  
> Huawei Technologies| 380| 3.1%  
> Facebook| 340| 2.8%  
> Code Aurora Forum| 316| 2.6%  
> (None)| 305| 2.5%  
> Mellanox| 271| 2.2%  
> Renesas Electronics| 270| 2.2%  
> ARM| 204| 1.7%  
> Bootlin| 169| 1.4%  
> (Unknown)| 158| 1.3%  
> linutronix| 153| 1.3%  
  
It can be instructive to compare these numbers to [those that were published for 2.6.24](<https://lwn.net/Articles/264440/>) in 2008. Many of the names in the left column were the same, though the ordering has changed — Andrew Morton had 1,679 non-author signoffs in 2.6.24. Many of the employer names are the same as well. But, in 2008, just over half of the non-author signoffs were made by developers working for two companies: Red Hat and the Linux Foundation. In 2018, those two organizations retain the top positions in the table, but one has to look at the top six companies to get up to 50% of the total. The process has been slow, but the concentration of maintainers has been dispersing over time. 

Finally, with regard to test and review credits, the numbers are: 

> Test and review credits in 4.18   
> ---  
> | Tested-by  
> ---  
> Andrew Bowers| 57| 7.7%  
> Nicholas Piggin| 43| 5.8%  
> Marek Szyprowski| 34| 4.6%  
> Arnaldo Carvalho de Melo| 21| 2.8%  
> Aaron Brown| 15| 2.0%  
> Angelo Dureghello| 14| 1.9%  
> Mathieu Malaterre| 14| 1.9%  
> Randy Dunlap| 13| 1.7%  
> Ard Biesheuvel| 13| 1.7%  
> Dmitry Osipenko| 12| 1.6%  
> Vijaya Kumar K| 12| 1.6%  
> Xiongfeng Wang| 12| 1.6%  
> Tomasz Nowicki| 12| 1.6%  
> Nguyen Viet Dung| 11| 1.5%  
> Jarkko Sakkinen| 8| 1.1%  
> Song Liu| 8| 1.1%  
> Geert Uytterhoeven| 7| 0.9%  
> | Reviewed-by  
> ---  
> Alex Deucher| 158| 3.2%  
> Rob Herring| 153| 3.1%  
> Geert Uytterhoeven| 115| 2.3%  
> Christian König| 104| 2.1%  
> Darrick J. Wong| 103| 2.1%  
> Christoph Hellwig| 99| 2.0%  
> David Sterba| 95| 1.9%  
> Neil Brown| 90| 1.8%  
> Laurent Pinchart| 87| 1.7%  
> Simon Horman| 84| 1.7%  
> Tony Cheng| 70| 1.4%  
> Andrew Morton| 61| 1.2%  
> Hawking Zhang| 55| 1.1%  
> Hannes Reinecke| 51| 1.0%  
> Brian Foster| 51| 1.0%  
> Chris Wilson| 46| 0.9%  
> Mika Kuoppala| 46| 0.9%  
  
The 4.18 kernel appears to be on track for an August 5 release, assuming no severe last-minute problems turn up. Once again, it will be the work of a huge community of developers, all of whom have managed to come together to improve this common resource. For all its faults, the kernel development community continues to function like a well-tuned machine, producing and integrating code at a pace that few other projects can match.
