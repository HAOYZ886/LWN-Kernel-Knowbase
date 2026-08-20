---
title: Who wrote 2.6.23
url: https://lwn.net/Articles/247582/
date: "September 11, 2007"
category: "Releases-2.6.23"
author: "By Jonathan Corbet September 11, 2007"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 11, 2007

While the 2.6.23 development cycle has not yet run its course, things are getting close enough to the end that it makes sense to start looking at the overall statistics for this release. As of this writing (shortly after 2.6.23-rc6 came out), just over 6,200 non-merge changesets had been added to the mainline kernel repository. These changesets came from 854 developers - a slightly smaller number than we saw for 2.6.22. Just over 350 of those developers contributed one single changeset. 

All told, the patches added almost 430,000 lines, but also removed 406,000 lines, meaning that the kernel grew by just under 23,000 lines - a relatively small number. That is partially a result of kernel hatcheteer Adrian Bunk's work: he removed the old SpeedStep code, a number of Open Sound System drivers, Rise CPU support, and more - a total of almost 73,000 lines removed. Jeff Garzik hacked out over 41,000 lines of network driver code, and Jens Axboe got rid of over 25,000 lines of code, mostly in the form of ancient CDROM drivers. 

Here is the list of the top contributors to 2.6.23, as counted by changesets merged and by lines of code changed: 

> Most active 2.6.23 developers  
> ---  
> | By changesets  
> ---  
> Ingo Molnar| 152| 2.5%  
> Ralf Baechle| 119| 1.9%  
> Trond Myklebust| 116| 1.9%  
> Paul Mundt| 111| 1.8%  
> David S. Miller| 107| 1.7%  
> Tejun Heo| 103| 1.7%  
> Al Viro| 95| 1.5%  
> Patrick McHardy| 93| 1.5%  
> Adrian Bunk| 92| 1.5%  
> FUJITA Tomonori| 91| 1.5%  
> Avi Kivity| 72| 1.2%  
> Andrew Morton| 71| 1.1%  
> Greg Kroah-Hartman| 62| 1.0%  
> Alan Cox| 58| 0.9%  
> David Brownell| 56| 0.9%  
> Jeff Garzik| 55| 0.9%  
> Christoph Hellwig| 54| 0.9%  
> Stephen Hemminger| 53| 0.9%  
> H. Peter Anvin| 52| 0.8%  
> Jesper Juhl| 52| 0.8%  
> | By changed lines  
> ---  
> Adrian Bunk| 73254| 11.0%  
> Jeff Garzik| 43253| 6.5%  
> Jens Axboe| 28004| 4.2%  
> Hirokazu Takata| 20399| 3.1%  
> Yoichi Yuasa| 18368| 2.8%  
> James Smart| 15626| 2.4%  
> Jeremy Fitzhardinge| 15398| 2.3%  
> David S. Miller| 14752| 2.2%  
> Matthew Wilcox| 14750| 2.2%  
> Christoph Hellwig| 14550| 2.2%  
> Rusty Russell| 9452| 1.4%  
> Imre Deak| 8925| 1.3%  
> Dan Williams| 8510| 1.3%  
> Ralf Baechle| 8345| 1.3%  
> Doug Thompson| 7310| 1.1%  
> Yoshihiro Shimoda| 6981| 1.1%  
> Marc St-Jean| 6888| 1.0%  
> Luca Olivetti| 6540| 1.0%  
> Cyrill Gorcunov| 6371| 1.0%  
> Latchesar Ionkov| 5375| 0.8%  
  
Ingo Molnar comes out on top of the changesets column by virtue of getting the CFS scheduler merged - then fixing it. Over half of his patches were accepted _after_ 2.6.23-rc1 came out. Ralf Baechle and Paul Mundt both contributed many changes to architecture-specific trees, Trond Myklebust did a lot of NFS work, and, while David Miller had a number of networking patches, the bulk of his changesets were in the architecture-specific (SPARC) trees. The figures on the "by changed lines" side are dominated by code removals (as described above); Jens Axboe also did a bunch of splice work and merged the "bsg" generic SCSI driver. Hirokazu Takata did a bunch of m32r architecture work. James Smart contributed a number of Fibre Channel changes and Jeremy Fitzhardinge merged the core Xen code. 

Once again, we have put some effort into associating patches with the companies that supported this work, with the results shown below. These results should always be taken as approximations; we believe that they are essentially correct, but patches do not come with Paid-for-by: headers, so a certain amount of guessing is always required. 

> Most active 2.6.23 employers  
> ---  
> | By changesets  
> ---  
> (Unknown)| 1180| 19.0%  
> Red Hat| 744| 12.0%  
> (None)| 559| 9.0%  
> IBM| 507| 8.2%  
> Novell| 421| 6.8%  
> Intel| 184| 3.0%  
> Oracle| 146| 2.4%  
> Renesas Technology| 134| 2.2%  
> MIPS Technologies| 119| 1.9%  
> NetApp| 116| 1.9%  
> (Consultant)| 103| 1.7%  
> Google| 99| 1.6%  
> NTT| 98| 1.6%  
> Sony| 93| 1.5%  
> Astaro| 93| 1.5%  
> Linux Foundation| 82| 1.3%  
> MontaVista| 81| 1.3%  
> SGI| 77| 1.2%  
> Qumranet| 72| 1.2%  
> QLogic| 62| 1.0%  
> | By lines changed  
> ---  
> (Unknown)| 111777| 16.9%  
> (None)| 99649| 15.0%  
> Red Hat| 84224| 12.7%  
> IBM| 39449| 5.9%  
> Oracle| 36205| 5.5%  
> Renesas Technology| 33152| 5.0%  
> HP| 18718| 2.8%  
> Tripeaks| 18567| 2.8%  
> Novell| 17990| 2.7%  
> Emulex| 15942| 2.4%  
> XenSource| 15426| 2.3%  
> Intel| 14962| 2.3%  
> Sony| 11945| 1.8%  
> Analog Devices| 10345| 1.6%  
> rPath| 9678| 1.5%  
> MIPS Technologies| 9171| 1.4%  
> Solid Boot Ltd.| 8937| 1.3%  
> MontaVista| 8065| 1.2%  
> PMC-Sierra| 6888| 1.0%  
> Astaro| 6687| 1.0%  
  
Red Hat retains its place at the top of the by-changesets list, though its percentage of changes has dropped a bit. By lines changed, developers known to be working on their own time (the "None" entry) beat out all corporate contributors. It is worth noting that much of lines-changed count for those developers is, in fact, lines removed. 

Looking at who added Signed-off-by: lines to patches is interesting, especially if one looks at signoffs added by people other than the author of the patch. In this way, one gets an idea of who the gatekeepers are. There is a slight change to how this calculation was done this time around: if a patch carried signoffs from both Linus Torvalds and Andrew Morton, Linus's was not counted. As a result of how the process works, everything that goes through Andrew gets a signoff from Linus; not counting those signoffs gives a more accurate picture of how the review was actually done. 

> Developers with the most signoffs (total 5653)  
> ---  
> Andrew Morton| 1247| 21.6%  
> Linus Torvalds| 397| 6.9%  
> David S. Miller| 381| 6.6%  
> Greg Kroah-Hartman| 329| 5.7%  
> Jeff Garzik| 287| 5.0%  
> James Bottomley| 264| 4.6%  
> Paul Mackerras| 223| 3.9%  
> Mauro Carvalho Chehab| 150| 2.6%  
> Len Brown| 128| 2.2%  
> Ralf Baechle| 122| 2.1%  
> Roland Dreier| 116| 2.0%  
> Andi Kleen| 113| 2.0%  
> Russell King| 101| 1.8%  
> Jaroslav Kysela| 100| 1.7%  
> John W. Linville| 70| 1.2%  
> Tony Luck| 65| 1.1%  
> Takashi Iwai| 63| 1.1%  
> Jens Axboe| 58| 1.0%  
> Martin Schwidefsky| 55| 1.0%  
> Ingo Molnar| 51| 0.9%  
  
One question which comes up sometimes is: how do these numbers look for specific parts of the kernel tree? Your editor duly hacked on his scripts to generate this sort of information. Here is a summary of the results - using the employer by-changesets numbers: 

> Employer changeset contributions by subsystem  
> ---  
> | /arch (1428 total)  
> ---  
> (Unknown)| 222| 15.5%  
> IBM| 198| 13.9%  
> Red Hat| 128| 9.0%  
> (None)| 108| 7.6%  
> Renesas Technology| 101| 7.1%  
> MIPS Technologies| 89| 6.2%  
> Sony| 55| 3.9%  
> Novell| 46| 3.2%  
> Intel| 46| 3.2%  
> rPath| 42| 2.9%  
> | /block (103 total)  
> ---  
> NTT| 27| 26.2%  
> Oracle| 15| 14.6%  
> (Unknown)| 10| 9.7%  
> IBM| 8| 7.8%  
> Red Hat| 6| 5.8%  
> (None)| 5| 4.9%  
> Miracle Linux| 4| 3.9%  
> Computer Consultants| 3| 2.9%  
> Novell| 3| 2.9%  
> Sony| 3| 2.9%  
> | /Documentation (241 total)  
> ---  
> (Unknown)| 66| 27.4%  
> Novell| 27| 11.2%  
> IBM| 19| 7.9%  
> Oracle| 19| 7.9%  
> (None)| 18| 7.5%  
> Intel| 16| 6.6%  
> Red Hat| 13| 5.4%  
> (Consultant)| 6| 2.5%  
> Freescale| 5| 2.1%  
> NEC| 4| 1.7%  
> | /drivers (2762 total)  
> ---  
> (Unknown)| 572| 20.7%  
> (None)| 356| 12.9%  
> Novell| 237| 8.6%  
> Red Hat| 236| 8.5%  
> IBM| 191| 6.9%  
> Intel| 130| 4.7%  
> (Consultant)| 68| 2.5%  
> NTT| 65| 2.4%  
> Qumranet| 63| 2.3%  
> QLogic| 61| 2.2%  
> | /fs (622 total)  
> ---  
> Red Hat| 107| 17.2%  
> Oracle| 80| 12.9%  
> NetApp| 74| 11.9%  
> (Unknown)| 72| 11.6%  
> Novell| 63| 10.1%  
> IBM| 56| 9.0%  
> Univ. of Michigan CITI| 35| 5.6%  
> SGI| 26| 4.2%  
> (Academia)| 19| 3.1%  
> SWsoft| 17| 2.7%  
> | /kernel (938 total)  
> ---  
> Red Hat| 259| 27.6%  
> (Unknown)| 129| 13.8%  
> IBM| 119| 12.7%  
> Renesas Technology| 52| 5.5%  
> (None)| 44| 4.7%  
> Novell| 36| 3.8%  
> MIPS Technologies| 31| 3.3%  
> Fujitsu| 30| 3.2%  
> Intel| 28| 3.0%  
> Linutronix| 27| 2.9%  
> | /mm (261 total)  
> ---  
> IBM| 38| 14.6%  
> (Unknown)| 38| 14.6%  
> Renesas Technology| 33| 12.6%  
> SGI| 29| 11.1%  
> Novell| 24| 9.2%  
> Google| 19| 7.3%  
> Red Hat| 13| 5.0%  
> (None)| 10| 3.8%  
> ARM| 7| 2.7%  
> igel| 6| 2.3%  
> | /net (833 total)  
> ---  
> (Unknown)| 178| 21.4%  
> Astaro| 92| 11.0%  
> Red Hat| 87| 10.4%  
> (None)| 71| 8.5%  
> IBM| 53| 6.4%  
> Linux Foundation| 48| 5.8%  
> NetApp| 47| 5.6%  
> Broadcom| 23| 2.8%  
> Intel| 18| 2.2%  
> HP| 17| 2.0%  
  
From these numbers, one might conclude that Red Hat developers are strong in the core kernel area, but they don't much like writing documentation. There is a lot of "hobbyist" participation in the driver subtree - not a particularly surprising result, since making a specific device work is a common itch for developers to scratch. Academics like to play with filesystems, as do, unsurprisingly, companies like Oracle and NetApp. 

Beyond being approximate, all of the numbers shown above will change a bit before the final 2.6.23 release, which is probably at least three weeks away. The patches which will be merged in the coming weeks should all be fixed, though, so the changes will, with any luck at all, be small. All told, 2.6.23 shows an active kernel development community with contributions from a large number of developers - and quite a few companies which employ them. The kernel remains a vibrant and alive base on which to build our free systems. 

(Thanks are due to Greg Kroah-Hartman for his contributions to the scripts used to generate these statistics).
