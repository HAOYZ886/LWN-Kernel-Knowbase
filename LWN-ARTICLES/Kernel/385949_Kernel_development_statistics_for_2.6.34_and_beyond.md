---
title: Kernel development statistics for 2.6.34 and beyond
url: https://lwn.net/Articles/385949/
date: "May 4, 2010"
category: "Releases-2.6.34"
author: "By Jonathan Corbet May 4, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 4, 2010

As of this writing, the current kernel prepatch is 2.6.34-rc6. A couple more prepatches are most likely due before the final release, but the number of changes to be found there should be small. In other words, 2.6.34 is close to its final form, so it makes sense to take a look at what has gone into this development cycle. In a few ways, 2.6.34 is an unusual kernel. 

This kernel has seen the addition of 9100 non-merge changesets from just over 1100 developers. That makes it somewhat smaller than its predecessors, as can be seen in this table: 

> Kernel| Patches| Devs  
> ---|---|---  
> 2.6.29 | 11,600 | 1170  
> 2.6.30 | 11,700 | 1130  
> 2.6.31 | 10,600 | 1150  
> 2.6.32 | 10,800 | 1230  
> 2.6.33 | 10,500 | 1150  
> 2.6.34 | 9,100 | 1110  
  
Developer participation in this development cycle was slightly lower than the usual, but not in any significant way. But, it seems, those developers had a bit less than usual that they needed to get done. One might be tempted to chalk that up to the shorter-than-usual merge window at the beginning of this cycle, but the fact of the matter is that Linus let enough new material in after 2.6.34-rc1 to make the merge window effectively as long as it ever was. 

The lists of the most active developers suggest that perhaps something else was going on: many of the developers who traditionally put large amounts of code into the kernel essentially sat out this cycle. 

> Most active 2.6.34 developers  
> ---  
> | By changesets  
> ---  
> Sage Weil| 212| 2.3%  
> Joe Perches| 169| 1.9%  
> Paul Mundt| 153| 1.7%  
> Uwe Kleine-Konig| 109| 1.2%  
> Mark Brown| 102| 1.1%  
> Ben Dooks| 96| 1.1%  
> Rafał Miłecki| 88| 1.0%  
> Dan Carpenter| 84| 0.9%  
> Alex Deucher| 83| 0.9%  
> H Hartley Sweeten| 80| 0.9%  
> Christoph Hellwig| 75| 0.8%  
> Johannes Berg| 74| 0.8%  
> Arnaldo Carvalho de Melo| 72| 0.8%  
> Bartlomiej Zolnierkiewicz| 64| 0.7%  
> David S. Miller| 63| 0.7%  
> Magnus Damm| 63| 0.7%  
> | By changed lines  
> ---  
> Sage Weil| 30233| 4.1%  
> Vladislav Zolotarov| 23119| 3.2%  
> Jarod Wilson| 19689| 2.7%  
> Mark Brown| 18513| 2.5%  
> Dimitris Michailidis| 13919| 1.9%  
> Manuel Lauss| 11831| 1.6%  
> Jorn Engel| 10810| 1.5%  
> Kukjin Kim| 10142| 1.4%  
> Alex Deucher| 9785| 1.3%  
> Amit Kumar Salecha| 9391| 1.3%  
> Michael Chan| 9336| 1.3%  
> Joe Perches| 8738| 1.2%  
> Paul Mundt| 8438| 1.2%  
> Haojian Zhuang| 8403| 1.1%  
> Magnus Damm| 8320| 1.1%  
> Matthias Benesch| 7739| 1.1%  
  
Sage Weil jumped to the top of both lists with the merger of the [Ceph distributed filesystem](<http://lwn.net/Articles/258516/>) and the subsequent bug-fixing activity. Joe Perches is the new king of the trivial patch; his work includes lots of checkpatch fixups, reworking print statements in network drivers, and no less than 37 patches implementing a rather belated cleanup of the floppy driver. Paul Mundt's work falls almost exclusively within his role as the maintainer of the Super-H architecture. Uwe Kleine-Konig works mostly within the ARM architecture code, and Mark Brown continues as the source of large amounts of sound driver and embedded processor code. 

On the "lines changed" side, Vladislav Zolotarov only contributed nine patches, all with the Broadcom NetXtreme II driver - but they included a large replacement of the in-tree firmware. Jarod Wilson's count was even smaller - three patches; he contributed the Broadcom Crystal HD driver to the staging tree. Dimitris Michailidis earned his place on the list with the new Chelsio Communications T4 Ethernet driver. 

Just over 180 employers were identified as having contributed to 2.6.34 - almost exactly the same as 2.6.33. With [the 2.6.33 summary](<http://lwn.net/Articles/373405/>), your editor suggested that Red Hat's position as the top contributor may soon be threatened; let's see how that prediction worked out for 2.6.34: 

> Most active 2.6.34 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1455| 16.0%  
> (Unknown)| 959| 10.5%  
> Red Hat| 934| 10.3%  
> Intel| 472| 5.2%  
> IBM| 354| 3.9%  
> Novell| 329| 3.6%  
> (Consultant)| 274| 3.0%  
> Nokia| 248| 2.7%  
> New Dream Network| 237| 2.6%  
> Renesas Technology| 188| 2.1%  
> Texas Instruments| 180| 2.0%  
> Pengutronix| 154| 1.7%  
> Oracle| 144| 1.6%  
> HP| 128| 1.4%  
> (Academia)| 125| 1.4%  
> Analog Devices| 123| 1.4%  
> AMD| 121| 1.3%  
> Fujitsu| 121| 1.3%  
> Marvell| 120| 1.3%  
> Wolfson Microelectronics| 101| 1.1%  
> | By lines changed  
> ---  
> Red Hat| 75235| 10.3%  
> (None)| 75160| 10.3%  
> (Unknown)| 67541| 9.2%  
> Broadcom| 56595| 7.7%  
> Intel| 33175| 4.5%  
> New Dream Network| 31501| 4.3%  
> (Consultant)| 29140| 4.0%  
> Novell| 24217| 3.3%  
> Wolfson Microelectronics| 20660| 2.8%  
> Renesas Technology| 16205| 2.2%  
> Chelsio| 13937| 1.9%  
> IBM| 13618| 1.9%  
> QLogic| 13182| 1.8%  
> MSC Vertriebs GmbH| 12545| 1.7%  
> Samsung| 12224| 1.7%  
> Marvell| 11914| 1.6%  
> Texas Instruments| 11228| 1.5%  
> Analog Devices| 11047| 1.5%  
> AMD| 10894| 1.5%  
> Nokia| 10217| 1.4%  
  
Looking at absolute numbers, Red Hat's contributions declined considerably from 2.6.33: 1223 changesets dropped to 934. Everybody else declined even further, though; Intel's changeset count was less than half of its value from 2.6.33. So Red Hat stays firmly at the top of the list. Many of the other companies on the list will be unsurprising, but readers may be forgiven for wondering about New Dream Network; that is a business co-founded by Ceph developer Sage Weil. 

If we look at non-author signoffs, we get a view of who the most active gatekeepers for the kernel are. Here, there are no surprises at all: 

> Most non-author signoffs  
> ---  
> | By developer  
> ---  
> David S. Miller| 1034| 13.0%  
> Greg Kroah-Hartman| 780| 9.8%  
> Andrew Morton| 546| 6.9%  
> John W. Linville| 546| 6.9%  
> Ingo Molnar| 348| 4.4%  
> Mauro Carvalho Chehab| 330| 4.2%  
> James Bottomley| 244| 3.1%  
> Dave Airlie| 150| 1.9%  
> Ralf Baechle| 144| 1.8%  
> H. Peter Anvin| 141| 1.8%  
> | By employer  
> ---  
> Red Hat| 2865| 36.1%  
> Novell| 1293| 16.3%  
> Intel| 565| 7.1%  
> Google| 547| 6.9%  
> (None)| 365| 4.6%  
> IBM| 289| 3.6%  
> (Consultant)| 194| 2.4%  
> Wind River| 145| 1.8%  
> Atomide| 130| 1.6%  
> Oracle| 128| 1.6%  
  
[Ten development cycles ago (2.6.24)](<http://lwn.net/Articles/264440/>), Andrew Morton was the most active gatekeeper, signing off on almost 1700 patches. His role as subsystem maintainer of last resort has declined over the years as more maintainers manage their own repositories and push patches directly to Linus. Speaking of Linus, he not only didn't make the list above, but he wasn't even close: his 71 signoffs put him in the 22nd position. Dave Airlie's position on the list is an indication of how much activity we are currently seeing in the graphics area. 

Once again, over 50% of the patches heading into the mainline kernel pass through the hands of somebody employed by either Red Hat or Novell. 

#### Looking forward

As of this writing, the opening of the 2.6.35 merge window can be expected sometime in the next 1-3 weeks. By the stated rules of the kernel development process, the bulk of the code intended for that merge window should already be in the linux-next tree. With that in mind, your editor pulled down the May 4 edition of linux-next to see what was up. There are currently 5144 non-merge changesets in that tree, representing 758 developers. The top contributors are: 

> Most active linux-next developers  
> ---  
> | By changesets  
> ---  
> Mauro Carvalho Chehab| 245| 4.8%  
> Eric Paris| 103| 2.0%  
> Alexander Graf| 84| 1.6%  
> Johannes Berg| 59| 1.1%  
> Juuso Oikarinen| 59| 1.1%  
> Jean-François Moine| 58| 1.1%  
> Luis R. Rodriguez| 58| 1.1%  
> Greg Kroah-Hartman| 52| 1.0%  
> Sujith| 52| 1.0%  
> Dan Carpenter| 51| 1.0%  
> | By changed lines  
> ---  
> Mauro Carvalho Chehab| 28743| 6.2%  
> Eliot Blennerhassett| 18429| 4.0%  
> Bob Beers| 11703| 2.5%  
> Luis R. Rodriguez| 10507| 2.3%  
> Steve Wise| 9447| 2.0%  
> Viresh Kumar| 9426| 2.0%  
> Jason Wessel| 8739| 1.9%  
> Sjur Braendeland| 8685| 1.9%  
> Stephen Rothwell| 7908| 1.7%  
> Matthias Benesch| 7739| 1.7%  
  
Mauro Carvalho Chehab has had a busy development cycle; beyond large amounts of Video4Linux work, he's jumped into the Nehelem EDAC (memory error detection and correction) code and is adding a new core for the management of infrared controllers. Eric Paris has done a bunch of security cleanup work; he also has the [fanotify subsystem](<http://lwn.net/Articles/360955/>) queued up. Eliot Blennerhassett, instead, has a single patch: a driver for AudioScience sound devices. 

It will be interesting to see how this list changes by the end of the 2.6.35 merge window. Even more interesting, arguably, will be the list of top non-author signoffs: 

> Most non-author signoffs (linux-next)  
> ---  
> | Mauro Carvalho Chehab| 651| 13.8%  
> John W. Linville| 507| 10.8%  
> David Miller| 462| 9.8%  
> Greg Kroah-Hartman| 411| 8.7%  
> Ingo Molnar| 170| 3.6%  
> Avi Kivity| 156| 3.3%  
> James Bottomley| 155| 3.3%  
> Reinette Chatre| 98| 2.1%  
> David Woodhouse| 93| 2.0%  
> Marcelo Tosatti| 72| 1.5%  
  
Subsystem maintainers are the folks who are charged with getting work into linux-next, so, if they all are doing their jobs, this list should not change much through the merge window. 

If the numbers do hold, 2.6.35 looks like another relatively subdued development cycle without huge amounts of exciting new stuff. Things do tend to change during the merge window, though, and surprises always show up from somewhere. So, even with resources like linux-next, it's hard to tell what the next development cycle will truly bring.
