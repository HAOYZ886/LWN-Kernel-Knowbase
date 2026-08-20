---
title: Statistics from the 4.17 kernel development cycle
url: https://lwn.net/Articles/756031/
date: "May 31, 2018"
category: "Releases-4.17"
author: "By Jonathan Corbet May 31, 2018"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 31, 2018

The 4.17 kernel appears to be on track for a June 3 release, barring an unlikely last-minute surprise. So the time has come for the usual look at some development statistics for this cycle. While 4.17 is a normal cycle for the most part, it does have one characteristic of note: it is the third kernel release ever to be smaller (in terms of lines of code) than its predecessor. 

The 4.17 kernel, as of just after 4.17-rc7, has brought in 13,453 non-merge changesets from 1,696 developers. Of those developers, 256 made their first contribution to the kernel in this cycle; that is the smallest number of first-time developers since 4.8 (which had 237). The changeset count is nearly equal to 4.16 (which had 13,630), but the developer count is down from the 1,774 seen in the previous cycle. 

Those developers added 690,000 lines of code, but removed 869,000, for a net reduction of nearly 180,000 lines. The main reason for the reduced line count, of course, is [the removal of eight unused architectures](<https://lwn.net/Articles/748074/>). It's worth noting that, even with that much code hacked out, 4.17 will still be a little bit larger than 4.15. 

The most active developers this time around were: 

> Most active 4.17 developers  
> ---  
> | By changesets  
> ---  
> Kuninori Morimoto| 245| 1.8%  
> Kirill Tkhai| 160| 1.2%  
> Arnd Bergmann| 148| 1.1%  
> Chris Wilson| 147| 1.1%  
> Colin Ian King| 133| 1.0%  
> Alexandre Belloni| 124| 0.9%  
> Rex Zhu| 122| 0.9%  
> Dominik Brodowski| 119| 0.9%  
> Christian König| 119| 0.9%  
> Mauro Carvalho Chehab| 106| 0.8%  
> Ajay Singh| 102| 0.8%  
> Ville Syrjälä| 100| 0.7%  
> Arnaldo Carvalho de Melo| 99| 0.7%  
> Geert Uytterhoeven| 94| 0.7%  
> Hans de Goede| 86| 0.6%  
> Masahiro Yamada| 83| 0.6%  
> Eric Dumazet| 77| 0.6%  
> Gustavo A. R. Silva| 72| 0.5%  
> Fabio Estevam| 72| 0.5%  
> Linus Walleij| 71| 0.5%  
> | By changed lines  
> ---  
> Arnd Bergmann| 315103| 22.9%  
> Jesper Nilsson| 100033| 7.3%  
> Greg Kroah-Hartman| 81362| 5.9%  
> Feifei Xu| 52509| 3.8%  
> David Howells| 40705| 3.0%  
> Tom St Denis| 32968| 2.4%  
> James Hogan| 31998| 2.3%  
> Anirudh Venkataramanan| 18937| 1.4%  
> Kuninori Morimoto| 16175| 1.2%  
> Corentin Labbe| 15265| 1.1%  
> John Crispin| 13188| 1.0%  
> Yasunari Takiguchi| 12983| 0.9%  
> Gilad Ben-Yossef| 12426| 0.9%  
> Greentime Hu| 11690| 0.8%  
> Rex Zhu| 11458| 0.8%  
> Erik Schmauss| 10980| 0.8%  
> Jacopo Mondi| 10842| 0.8%  
> Harry Wentland| 10198| 0.7%  
> Simon Horman| 9179| 0.7%  
> Eric Biggers| 8626| 0.6%  
  
Kuninori Morimoto contributed 245 patches, almost all concerned with a large renaming effort taking place in the ALSA sound driver subsystem. Kirill Thkai did a lot of work to increase parallelism in the network stack, Arnd Bergmann removed most of the old architecture code and did a lot of other cleanup (and year-2038) work throughout the kernel, Chris Wilson did a lot of work on the Intel i915 graphics driver, and Colin Ian King contributed a set of cleanup and typo-fixing patches. 

The lines-changed column is dominated by Bergmann and Jesper Nilsson (who removed the Cris architecture). Greg Kroah-Hartman deleted a bunch of staging code (including the venerable IRDA infrared driver stack), Feifei Xu added more AMD GPU definitions, and David Howells removed the mn10300 architecture and did a bunch of filesystem-level work. 

Work on 4.17 was supported by 241 companies that we were able to identify; the most active of those were: 

> Most active 4.17 employers  
> ---  
> | By changesets  
> ---  
> Intel| 1392| 10.3%  
> (None)| 977| 7.3%  
> Red Hat| 870| 6.5%  
> (Unknown)| 756| 5.6%  
> AMD| 754| 5.6%  
> IBM| 564| 4.2%  
> Renesas Electronics| 559| 4.2%  
> Linaro| 527| 3.9%  
> Google| 448| 3.3%  
> Mellanox| 405| 3.0%  
> SUSE| 400| 3.0%  
> Bootlin| 330| 2.5%  
> Samsung| 268| 2.0%  
> Oracle| 267| 2.0%  
> Huawei Technologies| 244| 1.8%  
> Odin| 232| 1.7%  
> ARM| 222| 1.7%  
> (Consultant)| 201| 1.5%  
> Canonical| 188| 1.4%  
> Code Aurora Forum| 181| 1.3%  
> | By lines changed  
> ---  
> Linaro| 338103| 24.6%  
> AMD| 138729| 10.1%  
> Axis Communications| 100396| 7.3%  
> Intel| 84613| 6.2%  
> Linux Foundation| 81678| 5.9%  
> Red Hat| 71152| 5.2%  
> Renesas Electronics| 42565| 3.1%  
> (None)| 35960| 2.6%  
> Imagination Technologies| 32000| 2.3%  
> IBM| 25841| 1.9%  
> ARM| 23906| 1.7%  
> (Unknown)| 22646| 1.6%  
> Google| 21390| 1.6%  
> BayLibre| 20931| 1.5%  
> Mellanox| 19081| 1.4%  
> Bootlin| 16256| 1.2%  
> (Consultant)| 15353| 1.1%  
> Sony| 14029| 1.0%  
> Fon| 13188| 1.0%  
> Samsung| 12823| 0.9%  
  
As usual, there are few surprises here. 

The Reviewed-by tag was created to credit those who review code prior to its merging into the kernel. The actual use of that tag is sporadic at best, making it a poor guide to who is actually performing code review. But it still can be worth a look (and people complain when we don't post it). So here is a list of the top credited reviewers, alongside the counts of non-author signoffs (which are also an indicator of patch review): 

> Most active 4.17 code reviewers  
> ---  
> | Reviewed-by tags  
> ---  
> Alex Deucher| 213| 4.5%  
> Rob Herring| 192| 4.1%  
> Tony Cheng| 123| 2.6%  
> Geert Uytterhoeven| 108| 2.3%  
> Andrew Morton| 102| 2.2%  
> Andy Shevchenko| 94| 2.0%  
> Christian König| 83| 1.8%  
> Chris Wilson| 69| 1.5%  
> Daniel Vetter| 64| 1.4%  
> Laurent Pinchart| 57| 1.2%  
> Sebastian Reichel| 56| 1.2%  
> Harry Wentland| 56| 1.2%  
> Johannes Thumshirn| 55| 1.2%  
> Hannes Reinecke| 55| 1.2%  
> Christoph Hellwig| 53| 1.1%  
> Guenter Roeck| 51| 1.1%  
> Simon Horman| 48| 1.0%  
> Darrick J. Wong| 45| 1.0%  
> David Sterba| 43| 0.9%  
> Ido Schimmel| 42| 0.9%  
> | Non-author signoffs  
> ---  
> David S. Miller| 1378| 10.9%  
> Greg Kroah-Hartman| 876| 7.0%  
> Alex Deucher| 640| 5.1%  
> Mark Brown| 537| 4.3%  
> Mauro Carvalho Chehab| 390| 3.1%  
> Andrew Morton| 335| 2.7%  
> Ingo Molnar| 318| 2.5%  
> Arnaldo Carvalho de Melo| 213| 1.7%  
> Michael Ellerman| 210| 1.7%  
> Herbert Xu| 209| 1.7%  
> Jens Axboe| 201| 1.6%  
> Martin K. Petersen| 198| 1.6%  
> Kalle Valo| 155| 1.2%  
> Thomas Gleixner| 153| 1.2%  
> David Sterba| 151| 1.2%  
> Jason Gunthorpe| 134| 1.1%  
> Jeff Kirsher| 133| 1.1%  
> Simon Horman| 128| 1.0%  
> Doug Ledford| 123| 1.0%  
> Shawn Guo| 121| 1.0%  
  
Unlike many maintainers, Alex Deucher applies a Reviewed-by tag to many patches that he applies to his repository, causing him to show up in both columns. Rob Herring has reviewed a wide range of patches centered mostly around device-tree bindings and related issues; these patches were generally applied by somebody else. Geert Uytterhoeven reviews patches from a fair variety of authors, but he is not normally the maintainer who applies them. 

Andrew Morton reviews far more code than he ever gets credit for. Until recently, though, that activity has not been reflected by Reviewed-by tags: he supplied exactly one in 2008, 14 in 2009, one in 2012, and one in 2015\. That changed in January of this year when he started adding Reviewed-by tags to many of the patches he applies to his own tree; this is [part of a broader effort](<https://lwn.net/Articles/756112/>) to ensure that all memory-management patches are reviewed. Morton understands what it means to truly review a patch, so each of those tags certainly indicates a real amount of work. 

Tony Cheng is an interesting and potentially different case. He is an employee of AMD and, seemingly without exception, his Reviewed-by tags are applied to patches from other AMD developers, and the reviews themselves do not appear on the public mailing lists. He also applies Reviewed-by tags to his own patches, which are relatively small and few in number ([example](<https://git.kernel.org/linus/3f7194bce26e>)). Reviewed-by tags from people working at the same company as the patch author are often looked at with suspicion by other developers, especially when the reviews happen behind closed doors. In truth, in-house reviews can be among the most rigorous and demanding — or they can be a rubber stamp. Either way, though, applying Reviewed-by tags to one's own patches is not how things are usually done. 

The signoff column, of course, shows which maintainers have been accepting the most patches. It does not guarantee that the maintainer has reviewed all of those patches before applying them, though maintainers should ensure that _somebody_ has reviewed them. In any case, there is certainly some amount of work implied by having signed off on a lot of patches. 

Use of Reviewed-by tags would appear to be increasing; over time, that may help to bring the amount of review work in the kernel into clearer focus. For now, though, its use remains both spotty and inconsistent; it's better than no data at all, but it's not even close to a complete picture. Overall, these numbers, like all in this article, are far from perfect metrics about who is really doing the work to keep the kernel project going. 

One thing that is clear from these numbers is that the kernel remains a busy place — one of the busiest software-development projects on the planet. It seems unlikely that things will slow down anytime soon.
