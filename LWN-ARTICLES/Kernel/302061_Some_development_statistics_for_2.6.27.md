---
title: Some development statistics for 2.6.27
url: https://lwn.net/Articles/302061/
date: "October 7, 2008"
category: "Releases-2.6.27"
author: "By Jonathan Corbet October 7, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 7, 2008

It's that time of the development cycle again: the 2.6.27 kernel, if not yet released by the time you read this, will be shortly. Various other LWN articles have looked at features found in this release; here we will look at where that code came from. 

As of 2.6.27-rc9, a total of 10,604 non-merge changesets had been added to the mainline for the 2.6.27 kernel; those patches added a total of 826,000 lines of code while removing 608,000, for a net growth of 217,000 lines. There were 1,109 developers who contributed to 2.6.27, representing over 150 employers. 376 of those developers contributed a single patch during this development cycle. 

The most active developers for 2.6.27 were: 

> Most active 2.6.27 developers  
> ---  
> | By changesets  
> ---  
> Ingo Molnar| 238| 2.2%  
> Bartlomiej Zolnierkiewicz| 235| 2.2%  
> Adrian Bunk| 221| 2.1%  
> David S. Miller| 206| 1.9%  
> Alan Cox| 196| 1.8%  
> Yinghai Lu| 192| 1.8%  
> Jeremy Fitzhardinge| 162| 1.5%  
> Tomas Winkler| 128| 1.2%  
> Ben Dooks| 120| 1.1%  
> Jean Delvare| 113| 1.1%  
> Steven Rostedt| 108| 1.0%  
> Harvey Harrison| 105| 1.0%  
> Pavel Emelyanov| 103| 1.0%  
> Thomas Gleixner| 101| 1.0%  
> Jean-Francois Moine| 89| 0.8%  
> Lennert Buytenhek| 88| 0.8%  
> Hans Verkuil| 81| 0.8%  
> Joerg Roedel| 81| 0.8%  
> Arnd Bergmann| 76| 0.7%  
> David Brownell| 75| 0.7%  
> | By changed lines  
> ---  
> Paul Mackerras| 138374| 12.1%  
> David Woodhouse| 44759| 3.9%  
> Jean-Francois Moine| 41157| 3.6%  
> Adrian Bunk| 35160| 3.1%  
> Artem Bityutskiy| 34545| 3.0%  
> Luis R. Rodriguez| 31825| 2.8%  
> Sam Ravnborg| 27443| 2.4%  
> Karsten Keil| 24674| 2.2%  
> Russell King| 22861| 2.0%  
> Eilon Greenstein| 19470| 1.7%  
> Alan Cox| 16957| 1.5%  
> Felipe Balbi| 16287| 1.4%  
> Kumar Gala| 14490| 1.3%  
> David Brownell| 12551| 1.1%  
> Ralf Baechle| 11057| 1.0%  
> Lennert Buytenhek| 9735| 0.9%  
> David S. Miller| 8621| 0.8%  
> Juergen Beisert| 8516| 0.7%  
> Steven Rostedt| 8455| 0.7%  
> Ben Dooks| 8399| 0.7%  
  
On the changeset side, Ingo Molnar ended up on top by virtue of the creation of large numbers of mostly x86-related changes, including a big subarchitecture reorganization; Ingo's count also includes the addition of ftrace, though much of that code was written by others. Bartlomiej Zolnierkiewicz continues to rework the old IDE layer, and Adrian Bunk, as always, energetically cleans up code all over the tree. David Miller's total includes the multiqueue networking code and a lot of other changes; Alan Cox did a lot of TTY work and big kernel lock removal. 

Your editor was disappointed to come in at #23, and, thus, off the bottom of the table. Time to send in some quick white space fixes. More seriously, though, it's worth noting that there are relatively few patches of the "trivial change" variety in the mix this time around. 

If we look at changed lines, Paul Mackerras comes out on top as the result of a single patch removing the obsolete ppc architecture. David Woodhouse reworked the management of firmware throughout the driver tree. Jean-François Moine brought the GSPCA webcam drivers into the tree, then put vast amounts of effort into cleaning them up. Artem Bityutskiy added the UBIFS flash filesystem, and Luis Rodriguez merged the ath9k wireless driver. 

If we look at the companies behind this work, we get the following results (note that, as always, these results are somewhat approximate): 

> Most active 2.6.27 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1925| 18.2%  
> Red Hat| 1405| 13.2%  
> (Unknown)| 921| 8.7%  
> IBM| 791| 7.5%  
> Intel| 605| 5.7%  
> Novell| 586| 5.5%  
> Movial| 234| 2.2%  
> SGI| 197| 1.9%  
> (Consultant)| 193| 1.8%  
> Sun| 184| 1.7%  
> XenSource| 165| 1.6%  
> Parallels| 157| 1.5%  
> Oracle| 148| 1.4%  
> Marvell| 143| 1.3%  
> Fujitsu| 138| 1.3%  
> AMD| 129| 1.2%  
> Renesas Technology| 125| 1.2%  
> linutronix| 121| 1.1%  
> Simtec| 119| 1.1%  
> (Academia)| 108| 1.0%  
> | By lines changed  
> ---  
> IBM| 207215| 18.1%  
> (None)| 129998| 11.4%  
> Red Hat| 109970| 9.6%  
> (Unknown)| 108878| 9.5%  
> Nokia| 52022| 4.5%  
> Novell| 49944| 4.4%  
> (Consultant)| 46529| 4.1%  
> Broadcom| 43438| 3.8%  
> Atheros| 38212| 3.3%  
> Movial| 35439| 3.1%  
> Intel| 32887| 2.9%  
> Freescale| 25511| 2.2%  
> SGI| 23444| 2.0%  
> Marvell| 20967| 1.8%  
> Renesas Technology| 15723| 1.4%  
> MIPS Technologies| 15701| 1.4%  
> Pengutronix| 13334| 1.2%  
> Atmel| 10786| 0.9%  
> Analog Devices| 10725| 0.9%  
> Sun| 9176| 0.8%  
  
There are not too many surprises in this table - in particular, the list of companies at the top tends not to change very much. That said, a few things are worthy of note. One is that Sun Microsystems has made its first appearance on this list. People complain about this company, but Sun's engineers have been quietly fixing things all over the tree. Broadcom is another company with a mixed reputation in the Linux community, but Broadcom is happy to provide support for some of its network adapters. Nokia's strong showing in the lines-changed table results primarily from the contribution of the UBIFS filesystem. 

The most welcome change, though, is the first appearance of Atheros on this list. Atheros is a company which has quickly moved from a position of complete non-cooperation to one of supporting all of its hardware in the mainline kernel. To say that this is an encouraging development would be an understatement. 

All told, the 2.6.27 development cycle shows that the process continues at full pace in a seemingly healthy state. Developers from all over the industry are all working together to make the kernel better for all. The number of companies which see participation in the process as being in their interest is growing, as is the number of developers who contribute patches. The Linux kernel, it seems, is in good shape.
