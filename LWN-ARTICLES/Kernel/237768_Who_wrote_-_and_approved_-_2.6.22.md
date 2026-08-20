---
title: "Who wrote - and approved - 2.6.22"
url: https://lwn.net/Articles/237768/
date: "June 11, 2007"
category: "Development model-Contributor statistics; Releases-2.6.22"
author: 
---

The 2.6.22 kernel is getting closer to its final state with its official release likely to happen near the end of this month. Patches are still being added to the mainline repository, but things have stabilized enough that it makes sense to take a look at where the code came from this time around. Accordingly, your editor has fixed up his scripts and cranked through the changesets added in this kernel development cycle. 

As of this writing, just over 6,000 changesets have been accepted for 2.6.22. Those patches were contributed by 885 different developers, added 494,000 lines, and deleted 241,000 other lines (without counting renames, which would otherwise increase both numbers by about 60,000 lines). That makes 2.6.22 a large change relative to its immediate predecessors: 

> Release| Developers| Changesets | Lines  
> added| Lines  
> removed  
> ---|---|---|---|---  
> 2.6.20| 741| 4983| 286,000 | 160,000  
> 2.6.21| 842| 5349| 343,000 | 199,000  
> 2.6.22-rc4+| 885| 6093 | 494,000| 241,000  
  
Here's the top contributors of those changes: 

> Most active 2.6.22 developers  
> ---  
> | By changesets  
> ---  
> David S. Miller| 175| 3.0%  
> Kristian Høgsberg| 109| 1.9%  
> Stephen Hemminger| 86| 1.5%  
> Arnaldo Carvalho de Melo| 82| 1.4%  
> Andrew Morton| 79| 1.3%  
> Stefan Richter| 79| 1.3%  
> Christoph Lameter| 77| 1.3%  
> Patrick McHardy| 76| 1.3%  
> Jean Delvare| 75| 1.3%  
> Dmitry Torokhov| 70| 1.2%  
> Stephen Rothwell| 68| 1.2%  
> Paul Mundt| 66| 1.1%  
> David Brownell| 65| 1.1%  
> Jeff Dike| 63| 1.1%  
> Alan Cox| 60| 1.0%  
> Andi Kleen| 59| 1.0%  
> Antonino Daplas| 58| 1.0%  
> Adrian Bunk| 58| 1.0%  
> Tejun Heo| 57| 1.0%  
> Russell King| 57| 1.0%  
> | By changed lines  
> ---  
> Bryan Wu| 77594| 12.9%  
> David Howells| 23310| 3.9%  
> Marcelo Tosatti| 22351| 3.7%  
> Patrick McHardy| 21746| 3.6%  
> Jiri Benc| 18328| 3.0%  
> Hans Verkuil| 13683| 2.3%  
> David S. Miller| 13595| 2.3%  
> Roland Dreier| 12247| 2.0%  
> Artem B. Bityutskiy| 12065| 2.0%  
> Kristian Høgsberg| 11153| 1.9%  
> Robert P. J. Day| 7554| 1.3%  
> Christoph Lameter| 7378| 1.2%  
> Andrew Victor| 6638| 1.1%  
> Mike Frysinger| 6313| 1.0%  
> David Brownell| 6033| 1.0%  
> Michael Chan| 5851| 1.0%  
> Andi Kleen| 5431| 0.9%  
> David Gibson| 5321| 0.9%  
> Nobuhiro Iwamatsu| 5296| 0.9%  
> Mark Fasheh| 4921| 0.8%  
  
Bryan Wu makes it to the top of the list of contributors (by lines changed) by virtue of being the person to contribute support for the [Blackfin](<http://www.analog.com/processors/blackfin/>) architecture. David Howells contributed the AF_RXRPC and AFS filesystem work; Marcelo Tosatti wrote the OLPC "Libertas" wireless driver, and Jiri Benc's name appears on the mac80211 stack. 

When broken down by employer, the (approximate, as always) numbers come out like this: 

> Most active 2.6.22 employers  
> ---  
> | By changesets  
> ---  
> (Unknown)| 1766| 30.2%  
> Red Hat| 720| 12.3%  
> IBM| 601| 10.3%  
> Novell| 411| 7.0%  
> (None)| 245| 4.2%  
> Intel| 203| 3.5%  
> Oracle| 127| 2.2%  
> (Consultant)| 119| 2.0%  
> Linux Foundation| 116| 2.0%  
> Google| 111| 1.9%  
> SGI| 93| 1.6%  
> Nokia| 83| 1.4%  
> Freescale| 80| 1.4%  
> Astaro| 76| 1.3%  
> XenSource| 56| 1.0%  
> MontaVista| 56| 1.0%  
> Qumranet| 55| 0.9%  
> HP| 53| 0.9%  
> QLogic| 52| 0.9%  
> Analog Devices| 49| 0.8%  
> | By lines changed  
> ---  
> (Unknown)| 130164| 21.6%  
> Red Hat| 104627| 17.4%  
> Analog Devices| 84561| 14.0%  
> Novell| 41366| 6.9%  
> IBM| 33629| 5.6%  
> Astaro| 22065| 3.7%  
> (None)| 20097| 3.3%  
> (Consultant)| 15403| 2.6%  
> Linutronix| 13585| 2.3%  
> Intel| 12288| 2.0%  
> Cisco| 12280| 2.0%  
> Oracle| 10482| 1.7%  
> Freescale| 10116| 1.7%  
> SGI| 8639| 1.4%  
> Nokia| 7328| 1.2%  
> SANPeople| 7045| 1.2%  
> Broadcom| 5952| 1.0%  
> MontaVista| 5810| 1.0%  
> Linux Foundation| 5746| 1.0%  
> Atmel| 5220| 0.9%  
  
One thing which jumps out here is that the amount of code contributed by developers known to be working on their own time has dropped; 2.6.22 will be one of the most corporate kernels yet. 

Looking at the developers who put Signed-off-by lines onto patches yields some interesting results. If one tabulates all 12,678 signoffs in 2.6.22, the results look like this: 

> Developers with the most signoffs (total 12678)  
> ---  
> Andrew Morton| 1415| 11.2%  
> Linus Torvalds| 1299| 10.2%  
> David S. Miller| 814| 6.4%  
> Paul Mackerras| 381| 3.0%  
> Jeff Garzik| 344| 2.7%  
> Andi Kleen| 252| 2.0%  
> Greg Kroah-Hartman| 236| 1.9%  
> Mauro Carvalho Chehab| 236| 1.9%  
> Stefan Richter| 210| 1.7%  
> Russell King| 189| 1.5%  
> James Bottomley| 176| 1.4%  
> Jaroslav Kysela| 145| 1.1%  
> Takashi Iwai| 131| 1.0%  
> Len Brown| 126| 1.0%  
> Kristian Høgsberg| 126| 1.0%  
> Patrick McHardy| 117| 0.9%  
> Jean Delvare| 110| 0.9%  
> Roland Dreier| 109| 0.9%  
> Antonino Daplas| 106| 0.8%  
> Dmitry Torokhov| 105| 0.8%  
  
All authors must sign off on their code. Additionally, any maintainer who passes a patch up toward the mainline adds a signoff indicating that he or she believes the code is legitimate and suitable for inclusion. If one excludes signoffs by the author of each patch, the remaining 7,000 signoffs are (almost) all by people through whom the code has passed (a few of them are by additional authors of the patch). Those adding non-author signoffs can thus be thought of as the gatekeepers through whom each patch must pass. Non-author signoffs break down like this: 

> Non-author signoffs (total 7028)  
> ---  
> Andrew Morton| 1336| 19.0%  
> Linus Torvalds| 1279| 18.2%  
> David S. Miller| 640| 9.1%  
> Paul Mackerras| 371| 5.3%  
> Jeff Garzik| 322| 4.6%  
> Greg Kroah-Hartman| 222| 3.2%  
> Mauro Carvalho Chehab| 216| 3.1%  
> Andi Kleen| 193| 2.7%  
> James Bottomley| 163| 2.3%  
> Jaroslav Kysela| 142| 2.0%  
> Russell King| 132| 1.9%  
> Stefan Richter| 131| 1.9%  
> Len Brown| 115| 1.6%  
> John W. Linville| 85| 1.2%  
> Roland Dreier| 85| 1.2%  
> Takashi Iwai| 79| 1.1%  
> Martin Schwidefsky| 54| 0.8%  
> David Woodhouse| 53| 0.8%  
> Ralf Baechle| 48| 0.7%  
> Antonino Daplas| 48| 0.7%  
  
In summary, 80% of the patches merged into the mainline kernel passed through the twenty developers listed above. One can take another step, and look at the number of non-author signoffs by employer: 

> Non-author signoffs by employer  
> ---  
> Google| 1338| 19.0%  
> Linux Foundation| 1281| 18.2%  
> Red Hat| 1246| 17.7%  
> Novell| 700| 10.0%  
> (Unknown)| 660| 9.4%  
> IBM| 553| 7.9%  
> (None)| 293| 4.2%  
> Intel| 193| 2.7%  
> SteelEye| 163| 2.3%  
> Cisco| 85| 1.2%  
> MIPS Technologies| 48| 0.7%  
> Nokia| 42| 0.6%  
> Astaro| 41| 0.6%  
> Analog Devices| 35| 0.5%  
> QLogic| 35| 0.5%  
> Cendio| 32| 0.5%  
> SGI| 28| 0.4%  
> NetApp| 28| 0.4%  
> (Consultant)| 23| 0.3%  
> Oracle| 22| 0.3%  
  
The bottom line: while Linux kernel development is a highly distributed activity, the work of several hundred developers is channeled through a surprisingly small number of individuals, and an even smaller number of companies on its way into the mainline.
