---
title: Developer statistics for 2.6.30
url: https://lwn.net/Articles/334721/
date: "May 27, 2009"
category: "Releases-2.6.30"
author: "By Jonathan Corbet May 27, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 27, 2009

As the 2.6.30 development cycle heads toward a close, it is natural to look back at what has been merged and where it came from. So here is LWN's traditional look at who wrote the code which went into the mainline this time around. 

Once again, 2.6.30 was a large development cycle; it saw the incorporation (through just after 2.6.30-rc7) of 11,733 non-merge changesets from 1125 developers. The number of changesets exceeds 2.6.29, but the number of developers falls just short of the 1166 seen last time around. Those developers added 1.14 million lines of code this time around, while taking out 513,000, for a net growth of 624,000 lines. 

The individual developer statistics for 2.6.30 look like: 

> Most active 2.6.30 developers  
> ---  
> | By changesets  
> ---  
> Ingo Molnar| 324| 2.8%  
> Bill Pemberton| 227| 1.9%  
> Stephen Hemminger| 204| 1.7%  
> Hans Verkuil| 199| 1.7%  
> Takashi Iwai| 188| 1.6%  
> Bartlomiej Zolnierkiewicz| 186| 1.6%  
> Steven Rostedt| 179| 1.5%  
> Greg Kroah-Hartman| 150| 1.3%  
> Jeremy Fitzhardinge| 125| 1.1%  
> Mark Brown| 107| 0.9%  
> Jaswinder Singh Rajput| 105| 0.9%  
> Rusty Russell| 100| 0.9%  
> Tejun Heo| 98| 0.8%  
> Johannes Berg| 98| 0.8%  
> Hannes Eder| 88| 0.8%  
> Michal Simek| 85| 0.7%  
> Luis R. Rodriguez| 85| 0.7%  
> Sujith| 85| 0.7%  
> David Howells| 80| 0.7%  
> Yinghai Lu| 78| 0.7%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 120353| 9.0%  
> ADDI-DATA GmbH| 43420| 3.3%  
> Mithlesh Thukral| 42424| 3.2%  
> Alex Deucher| 26576| 2.0%  
> David Schleef| 25905| 1.9%  
> David Woodhouse| 24636| 1.8%  
> Ramkrishna Vepa| 23495| 1.8%  
> Lior Dotan| 22506| 1.7%  
> Eric Moore| 22266| 1.7%  
> Eilon Greenstein| 18399| 1.4%  
> Jaswinder Singh Rajput| 18168| 1.4%  
> Hans Verkuil| 18048| 1.4%  
> David Howells| 17941| 1.3%  
> Andy Grover| 16355| 1.2%  
> Michal Simek| 15827| 1.2%  
> Sri Deevi| 15514| 1.2%  
> Frank Mori Hess| 15450| 1.2%  
> Ben Hutchings| 15031| 1.1%  
> Ingo Molnar| 13876| 1.0%  
> Bill Pemberton| 13817| 1.0%  
  
On the changesets side, Ingo Molnar is at the top of the list this time around; as usual, he created a vast number of patches - about five per day \- in the x86 architecture code, ftrace, and beyond. Bill Pemberton is perhaps better known as the maintainer of the Elm mail client; he did a lot of cleanup work with the COMEDI drivers in the -staging tree. The bulk of Stephen Hemminger's work involved converting network drivers to the new `net_device_ops` API. Hans Verkuil continues to improve the Video4Linux2 framework and associated drivers, and Takashi Iwai continues to generate a lot of patches as the ALSA maintainer. 

Linus kicked off the 2.6.30 development cycle by [noting](<http://lwn.net/Articles/327772/>) that about one third of the changes in 2.6.30-rc1 were "crap." So, unsurprisingly, the top three entries in the "by changed lines" column all got there through the addition of -staging drivers. Alex Deucher added Radeon R6xx/R7xx support; many of his "changed lines" were associated microcode firmware. And David Schleef added another set of drivers to the -staging tree. 

Contributions to 2.6.30 could be traced back to some 190 employers. Looking at the most-active employer information, we see: 

> Most active 2.6.30 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1970| 16.8%  
> Red Hat| 1305| 11.1%  
> (Unknown)| 1184| 10.1%  
> Intel| 855| 7.3%  
> Novell| 832| 7.1%  
> IBM| 630| 5.4%  
> (Consultant)| 293| 2.5%  
> Atheros Communications| 262| 2.2%  
> Oracle| 252| 2.1%  
> University of Virginia| 227| 1.9%  
> Fujitsu| 217| 1.8%  
> Vyatta| 204| 1.7%  
> Renesas Technology| 152| 1.3%  
> NTT| 121| 1.0%  
> MontaVista| 115| 1.0%  
> HP| 107| 0.9%  
> Wolfson Microelectronics| 105| 0.9%  
> (Academia)| 102| 0.9%  
> Nokia| 98| 0.8%  
> XenSource| 91| 0.8%  
> | By lines changed  
> ---  
> (Unknown)| 181413| 13.6%  
> Novell| 164229| 12.3%  
> (None)| 118095| 8.9%  
> Intel| 86060| 6.5%  
> Red Hat| 73954| 5.5%  
> LinSysSoft Technologies| 64798| 4.9%  
> ADDI-DATA GmbH| 43420| 3.3%  
> SofaWare| 39245| 2.9%  
> Broadcom| 31956| 2.4%  
> AMD| 28364| 2.1%  
> Entropy Wave| 25905| 1.9%  
> IBM| 25702| 1.9%  
> Oracle| 25588| 1.9%  
> NTT| 25235| 1.9%  
> Neterion| 23495| 1.8%  
> LSI Logic| 22304| 1.7%  
> Atheros Communications| 21627| 1.6%  
> (Consultant)| 19209| 1.4%  
> Freescale| 16139| 1.2%  
> PetaLogix| 15846| 1.2%  
  
These numbers are somewhat similar to those seen in previous development cycles. There are a few unfamiliar companies here; they are pretty much all present as a result of contributions to -staging. It is interesting to note that Atheros and Broadcom, once known as uncooperative companies, are increasing their contributions over time. 

Your editor has not looked at signoff statistics for the last few cycles. The interesting thing to be found in Signed-off-by tags is an indication of who the gatekeepers to the kernel are. Especially if one disregards signoffs by the author of each patch, what remains is (mostly) the signoffs of subsystem maintainers who approved the patches for merging. For 2.6.30, these numbers look like this: 

> Top non-author signoffs in 2.6.30  
> ---  
> | Individuals  
> ---  
> David S. Miller| 1216| 12.1%  
> John W. Linville| 865| 8.6%  
> Ingo Molnar| 836| 8.3%  
> Greg Kroah-Hartman| 797| 7.9%  
> Mauro Carvalho Chehab| 784| 7.8%  
> Andrew Morton| 660| 6.6%  
> James Bottomley| 250| 2.5%  
> Linus Torvalds| 219| 2.2%  
> Len Brown| 189| 1.9%  
> Takashi Iwai| 165| 1.6%  
> Jeff Kirsher| 145| 1.4%  
> Russell King| 127| 1.3%  
> H. Peter Anvin| 120| 1.2%  
> Mark Brown| 115| 1.1%  
> Jesse Barnes| 111| 1.1%  
> Benjamin Herrenschmidt| 111| 1.1%  
> Reinette Chatre| 104| 1.0%  
> Martin Schwidefsky| 95| 0.9%  
> Avi Kivity| 91| 0.9%  
> Paul Mundt| 89| 0.9%  
> | Employers  
> ---  
> Red Hat| 4264| 42.4%  
> Novell| 1386| 13.8%  
> Intel| 951| 9.5%  
> Google| 660| 6.6%  
> (None)| 408| 4.1%  
> IBM| 378| 3.8%  
> Linux Foundation| 219| 2.2%  
> (Consultant)| 166| 1.6%  
> (Unknown)| 127| 1.3%  
> Wolfson Microelectronics| 115| 1.1%  
> Renesas Technology| 92| 0.9%  
> Marvell| 91| 0.9%  
> Atomide| 81| 0.8%  
> Oracle| 80| 0.8%  
> Astaro| 65| 0.6%  
> Freescale| 63| 0.6%  
> Cisco| 61| 0.6%  
> Analog Devices| 60| 0.6%  
> Univ. of Michigan CITI| 59| 0.6%  
> Panasas| 58| 0.6%  
  
Signoffs have always been more concentrated than contributions in general. Still, one wonders how David Miller manages to approve a solid twenty patches every day. On the employer side, things are more concentrated than ever; over half of the patches going into the kernel go through the hands of a developer at Red Hat or Novell. Developers, it seems, work for a great many companies, but subsystem maintainers gravitate toward a small handful of firms. 

All told, the picture remains one of a well-oiled, fast-moving development process. We also see a picture of a -staging tree which is growing at a tremendous rate; your editor is tempted to exclude -staging patches from future reports if the rate does not slow somewhat. Even without -staging, though, a _lot_ of work is being done on the kernel, with the participation of a large group of developers, and it doesn't look like it will be slowing down anytime soon. 

**Postscript** : Jan Engelhardt sent your editor a pointer to [a short script](<http://dev.medozas.de/gitweb.cgi?p=hxtools;a=blob;f=libexec/git-blame-stats;hb=master>) which, through use of the `git blame` command, tallies up the "ownership" of every line in the kernel. The top results for 2.6.30-rc7 look like this: 

> Who last touched kernel code lines  
> ---  
> Lines| Pct| Who  
> 4063723| 35.17% | Linus Torvalds  
> 464021| 4.02% | Greg Kroah-Hartman  
> 94200| 0.82% | David Howells  
> 86031| 0.74% | David S. Miller  
> 82608| 0.71% | Luis R. Rodriguez  
> 72200| 0.62% | Bryan Wu  
> 70128| 0.61% | Takashi Iwai  
> 66859| 0.58% | Ralf Baechle  
> 55785| 0.48% | Hans Verkuil  
> 54069| 0.47% | Paul Mundt  
> 54007| 0.47% | Kumar Gala  
> 53288| 0.46% | David Brownell  
> 51640| 0.45% | Russell King  
> 50611| 0.44% | Paul Mackerras  
> 49499| 0.43% | Andrew Victor  
> 49347| 0.43% | Mauro Carvalho Chehab  
> 49256| 0.43% | Alan Cox  
> 47305| 0.41% | Mikael Starvik  
> 47040| 0.41% | Ben Dooks  
> 44307| 0.38% | Benjamin Herrenschmidt  
  
Linus shows a high ownership because he was the initial committer at the beginning of the git era. To a rough approximation, one can conclude that approximately one third of the code in the kernel has not been touched since that time. There are other interesting things which can be done with line-level statistics; your editor plans to explore this idea some in the future.
