---
title: Development statistics for 2.6.28
url: https://lwn.net/Articles/312074/
date: "December 18, 2008"
category: "Releases-2.6.28"
author: "By Jonathan Corbet December 18, 2008"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 18, 2008

As of this writing, the 2.6.28 kernel is getting quite close to its final release. The flow of patches into the mainline repository has slowed to a trickle. So it become appropriate to look at what was done in this development cycle, and where all that code came from. 

In these articles, your editor routinely forgets to thank Greg Kroah-Hartman, who continues to do a lot of work to ensure that these statistics are at least moderately accurate. So we'll get that taken care of at the outset: thanks, Greg! 

The 2.6.28 development cycle has seen the incorporation of just under 9,000 changesets; that makes it a bit smaller in this regard than 2.6.27 (10,600) or 2.6.26 (10,100). The development base broadened, though; 1,262 developers have contributed to 2.6.28, more than has been seen with its predecessors. Those developers added 769,000 lines of code while removing 285,000, for a net growth of 484,000 lines - a relatively large amount. Much of that growth came by way of a single developer, as we will see below. 

In recent development cycles, some 25% of the patches merged were accepted after the close of the merge window. Linus Torvalds has been making sounds about tightening the criteria for patches during the stabilization period, to the point that they would have to address known regressions to be accepted. A look at 2.6.28, though, shows that 1835 patches (so far) have gone in since 2.6.28-rc1. At 20% of the total, the patch flow rate during the stabilization period has fallen - but not by much. 

So where did these patches come from? Here's the top twenty contributors to 2.6.28: 

> Most active 2.6.28 developers  
> ---  
> | By changesets  
> ---  
> David S. Miller| 239| 2.7%  
> Yinghai Lu| 200| 2.2%  
> Al Viro| 154| 1.7%  
> Bartlomiej Zolnierkiewicz| 150| 1.7%  
> Alexey Dobriyan| 121| 1.3%  
> Paul Mundt| 117| 1.3%  
> Ingo Molnar| 109| 1.2%  
> Gerrit Renker| 109| 1.2%  
> Russell King| 91| 1.0%  
> Johannes Berg| 91| 1.0%  
> Steven Rostedt| 85| 0.9%  
> Alan Cox| 84| 0.9%  
> Takashi Iwai| 83| 0.9%  
> Tejun Heo| 75| 0.8%  
> Harvey Harrison| 75| 0.8%  
> Mark Brown| 75| 0.8%  
> Suresh Siddha| 73| 0.8%  
> Joerg Roedel| 72| 0.8%  
> Hans Verkuil| 71| 0.8%  
> Eric Miao| 70| 0.8%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 127848| 14.9%  
> Inaky Perez-Gonzalez| 24084| 2.8%  
> Mark Brown| 17714| 2.1%  
> Joseph Chan| 15749| 1.8%  
> Pavel Machek| 15529| 1.8%  
> David S. Miller| 15368| 1.8%  
> Herbert Xu| 13309| 1.5%  
> Yinghai Lu| 12861| 1.5%  
> Paul Mundt| 10088| 1.2%  
> Magnus Damm| 10077| 1.2%  
> James Smart| 8103| 0.9%  
> Gerrit Renker| 7536| 0.9%  
> Johannes Berg| 7196| 0.8%  
> Bartlomiej Zolnierkiewicz| 7182| 0.8%  
> Eric Miao| 7130| 0.8%  
> Ron Mercer| 7093| 0.8%  
> Michael Buesch| 6475| 0.8%  
> Nick Kossifidis| 6380| 0.7%  
> David Vrabel| 6357| 0.7%  
> Adrian Bunk| 6289| 0.7%  
  
On the changesets side, David Miller contributes a lot of work to the network stack, but the bulk of his changes this time around are to the SPARC architecture code. Yinghai Lu is a constant source of x86 architecture patches. Al Viro returns to the list with a lot of cleanup work in the VFS code, user-mode Linux, and beyond. Bartlomiej Zolnierkiewicz continues to clean up the legacy IDE code, despite the fact that its user base is shrinking. And Alexey Dobriyan contributed work in a number of areas, with the bulk of it being in the netfilter subsystem and `/proc`. 

When looking at changed lines, one gets the sense that Greg Kroah-Hartman has been rather busy this time around. As it happens, Greg did not actually write most of that code; the bulk of it came in with the addition of the -staging tree. It seems that Greg, the self-named "maintainer of crap," has acquired substantial amounts of it. Inaky Perez-Gonzalez was the source of the patches adding support for ultrawideband radio and wireless USB. Expect to see him show up again soon; he is now working to get the WIMAX subsystem into the kernel. Mark Brown added drivers for a number of Wolfson Micro devices. Joseph Chan contributed the VIA framebuffer driver, and Pavel Machek added a handful of miscellaneous drivers. 

So who paid for this work to be done? The 2.6.28 employer table looks like this: 

> Most active 2.6.28 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1683| 18.8%  
> Red Hat| 1101| 12.3%  
> (Unknown)| 790| 8.8%  
> Intel| 654| 7.3%  
> IBM| 526| 5.9%  
> Novell| 460| 5.1%  
> (Consultant)| 227| 2.5%  
> Oracle| 206| 2.3%  
> Sun| 203| 2.3%  
> Renesas Technology| 169| 1.9%  
> AMD| 158| 1.8%  
> Parallels| 152| 1.7%  
> Marvell| 134| 1.5%  
> (Academia)| 131| 1.5%  
> Analog Devices| 122| 1.4%  
> HP| 120| 1.3%  
> University of Aberdeen| 109| 1.2%  
> Fujitsu| 106| 1.2%  
> Nokia| 97| 1.1%  
> Freescale| 87| 1.0%  
> | By lines changed  
> ---  
> Novell| 159527| 18.6%  
> (None)| 119373| 13.9%  
> (Unknown)| 78785| 9.2%  
> Red Hat| 67972| 7.9%  
> Intel| 64108| 7.5%  
> IBM| 31289| 3.6%  
> Renesas Technology| 24900| 2.9%  
> Sun| 19926| 2.3%  
> (Consultant)| 19605| 2.3%  
> Wolfson Micro| 17697| 2.1%  
> VIA| 17210| 2.0%  
> Marvell| 14108| 1.6%  
> Freescale| 12693| 1.5%  
> Oracle| 12101| 1.4%  
> Analog Devices| 10170| 1.2%  
> University of Aberdeen| 9969| 1.2%  
> Emulex| 8112| 0.9%  
> Nokia| 7744| 0.9%  
> QLogic| 7676| 0.9%  
> Atmel| 6885| 0.8%  
  
In general, the employer tables tend not to change too much from one development cycle to the next. Greg's staging tree work did put Novell at the top of the lines-changed column, despite the fact that this work did not originate at Novell. As always, one needs to bear in mind that these numbers are approximate. 

One welcome change is the first-time appearance of VIA. It appears that this company is truly getting serious about supporting Linux, and that can only be a good thing. 

Writing all this code is important, but so is reviewing, testing, and reporting bugs. Continuing with a relatively new tradition, we'll look at who shows up in patch tags indicating this kind of participation, starting with the reviewers: 

> Developers with the most reviews (total 83)  
> ---  
> James Morris| 12| 14.5%  
> Rene Herman| 12| 14.5%  
> Matthew Wilcox| 6| 7.2%  
> KOSAKI Motohiro| 5| 6.0%  
> Richard Genoud| 4| 4.8%  
> Tomas Winkler| 3| 3.6%  
> Paul E. McKenney| 3| 3.6%  
> Mingming Cao| 2| 2.4%  
> Michael Krufky| 2| 2.4%  
> KAMEZAWA Hiroyuki| 2| 2.4%  
> Pekka Enberg| 2| 2.4%  
> Daisuke Nishimura| 2| 2.4%  
> Christoph Lameter| 2| 2.4%  
> Balbir Singh| 2| 2.4%  
> Julius Volz| 2| 2.4%  
  
At this point, we are seeing about one Reviewed-by tag for every 100 changes going into the mainline repository. Fortunately, the review situation is not quite that bad; most reviewers simply do not provide these tags for the patches they look at. 

The numbers for bug reporting and patch testing look like this: 

> Most credited 2.6.28 testers  
> ---  
> | `Reported-by` credits  
> ---  
> Adrian Bunk| 5| 2.6%  
> Randy Dunlap| 4| 2.1%  
> Arjan van de Ven| 3| 1.5%  
> Ingo Molnar| 3| 1.5%  
> Stephen Rothwell| 3| 1.5%  
> Robert P. J. Day| 3| 1.5%  
> Stephane Eranian| 3| 1.5%  
> Daniel Marjamaki| 3| 1.5%  
> Rafael J. Wysocki| 2| 1.0%  
> Yinghai Lu| 2| 1.0%  
> Venki Pallipadi| 2| 1.0%  
> Eric Dumazet| 2| 1.0%  
> Carlos R. Mafra| 2| 1.0%  
> Wu Fengguang| 2| 1.0%  
> Zoltan Borbely| 2| 1.0%  
> Andy Wettstein| 2| 1.0%  
> Steven Noonan| 2| 1.0%  
> Alexander Beregalov| 2| 1.0%  
> Andrew Morton| 2| 1.0%  
> Alexey Dobriyan| 2| 1.0%  
> Heiko Carstens| 2| 1.0%  
> Jiri Slaby| 2| 1.0%  
> Sergei Shtylyov| 2| 1.0%  
> Johannes Weiner| 2| 1.0%  
> Mike Galbraith| 2| 1.0%  
> Hideo Saito| 2| 1.0%  
> Zvonimir Rakamaric| 2| 1.0%  
> Rik Theys| 2| 1.0%  
> Andreas Steffen| 2| 1.0%  
> Vegard Nossum| 2| 1.0%  
> | `Tested-by:` credits  
> ---  
> Ingo Molnar| 5| 2.9%  
> Dirk Teurlings| 5| 2.9%  
> Peter van Valderen| 5| 2.9%  
> Nicolas Pitre| 4| 2.3%  
> Matt Helsley| 4| 2.3%  
> Christian Borntraeger| 3| 1.7%  
> Rafael J. Wysocki| 3| 1.7%  
> Riku Voipio| 3| 1.7%  
> Byron Bradley| 3| 1.7%  
> Tim Ellis| 3| 1.7%  
> Kamalesh Babulal| 3| 1.7%  
> Alan Jenkins| 3| 1.7%  
> Robert Jarzmik| 3| 1.7%  
> Martyn Welch| 3| 1.7%  
> Takashi Iwai| 2| 1.2%  
> Badari Pulavarty| 2| 1.2%  
> Jeff Moyer| 2| 1.2%  
> Eric Dumazet| 2| 1.2%  
> Jesper Dangaard Brouer| 2| 1.2%  
> Ramon Casellas| 2| 1.2%  
> Markus Trippelsdorf| 2| 1.2%  
> Sitsofe Wheeler| 2| 1.2%  
> Andrey Borzenkov| 2| 1.2%  
  
In each case, everybody with at least two credits was listed. The good news is that, while there's certainly some familiar names on that list, we are also seeing appearances by people who are not known as kernel developers. There really is a testing community out there which includes more than just developers. Your editor suspects that we still are not doing a very good job of crediting them for their work, but this convention is relatively new and we can still hope for progress in this direction. To that end, the developers who are crediting reporters and testers are: 

> Developers giving credits in 2.6.28  
> ---  
> | `Reported-by` credits  
> ---  
> Jiri Kosina| 9| 4.6%  
> Ingo Molnar| 8| 4.1%  
> Adrian Bunk| 7| 3.6%  
> Bartlomiej Zolnierkiewicz| 6| 3.1%  
> Linus Torvalds| 6| 3.1%  
> Peter Zijlstra| 6| 3.1%  
> Markus Metzger| 6| 3.1%  
> Randy Dunlap| 5| 2.6%  
> Andrew Morton| 5| 2.6%  
> Yinghai Lu| 4| 2.1%  
> Venki Pallipadi| 4| 2.1%  
> Jiri Slaby| 4| 2.1%  
> Suresh Siddha| 4| 2.1%  
> Roland Dreier| 4| 2.1%  
> Patrick McHardy| 4| 2.1%  
> Mark Brown| 4| 2.1%  
> Takashi Iwai| 3| 1.5%  
> Steven Rostedt| 3| 1.5%  
> Stefan Richter| 3| 1.5%  
> Paul Mundt| 3| 1.5%  
> Thomas Gleixner| 3| 1.5%  
> Dmitry Torokhov| 3| 1.5%  
> | `Tested-by:` credits  
> ---  
> Lennert Buytenhek| 22| 12.8%  
> Takashi Iwai| 6| 3.5%  
> Rafael J. Wysocki| 5| 2.9%  
> Linus Torvalds| 5| 2.9%  
> Alan Stern| 5| 2.9%  
> Alexey Starikovskiy| 5| 2.9%  
> Henrik Rydberg| 5| 2.9%  
> Matt Helsley| 4| 2.3%  
> KAMEZAWA Hiroyuki| 4| 2.3%  
> Russell King| 4| 2.3%  
> Patrick McHardy| 4| 2.3%  
> Paul Mundt| 3| 1.7%  
> Jens Axboe| 3| 1.7%  
> Theodore Tso| 3| 1.7%  
> Bartlomiej Zolnierkiewicz| 3| 1.7%  
> Jean Delvare| 3| 1.7%  
> Thomas Gleixner| 3| 1.7%  
> David Brownell| 3| 1.7%  
> FUJITA Tomonori| 3| 1.7%  
  
A quick grep shows that the number of Reported-by and Tested-by tags in patches was almost exactly the same over the 2.6.27 and 2.6.28 development cycles. Given the smaller number of patches in 2.6.28, this indicates that a slightly higher percentages of patches are now carrying those tags. Emphasis on "slightly" is in order, though; we are, for the most part, still not crediting a great many people who have helped to get 2.6.28 into shape.
