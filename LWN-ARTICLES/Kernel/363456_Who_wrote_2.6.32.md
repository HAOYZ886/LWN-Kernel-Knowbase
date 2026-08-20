---
title: Who wrote 2.6.32
url: https://lwn.net/Articles/363456/
date: "November 24, 2009"
category: "Releases-2.6.32"
author: "By Jonathan Corbet November 24, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 24, 2009

As of this writing, the 2.6.32 appears poised for a release right around the beginning of December. That can only mean that the time has come to look at the code which has gone into this kernel and where it came from. It has been another active cycle, with a lot of changes making it into the mainline. 

In particular, as of this writing (shortly after the 2.6.32-rc8 release), 2.6.32 is the result of 10,767 non-merge changesets sent in by 1,229 developers. These changes added a total of 1.17 million lines, while removing 611,000 lines, for a net growth of 559,000 lines of code. According to [Rafael Wysocki's regression reports](<http://lwn.net/Articles/363280/>), this development cycle introduced a total of 86 regressions into the kernel - slightly fewer than we saw for 2.6.31. As of that posting, the number of unresolved regressions was shrinking quickly, with 25 of them still without a resolution. 

So who added all those ~~regressions~~ lines of code? The statistics for this cycle look like this: 

> Most active 2.6.32 developers  
> ---  
> | By changesets  
> ---  
> Greg Kroah-Hartman| 202| 1.9%  
> Johannes Berg| 180| 1.7%  
> Bartlomiej Zolnierkiewicz| 164| 1.5%  
> Mark Brown| 154| 1.4%  
> Paul Mundt| 139| 1.3%  
> Takashi Iwai| 139| 1.3%  
> Alan Cox| 129| 1.2%  
> Roel Kluin| 115| 1.1%  
> Luis R. Rodriguez| 105| 1.0%  
> Dan Williams| 86| 0.8%  
> Tejun Heo| 84| 0.8%  
> Herbert Xu| 81| 0.8%  
> Peter Zijlstra| 80| 0.7%  
> Ingo Molnar| 77| 0.7%  
> Julia Lawall| 77| 0.7%  
> Steven Rostedt| 73| 0.7%  
> Magnus Damm| 72| 0.7%  
> Joe Perches| 71| 0.7%  
> Joerg Roedel| 70| 0.7%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 174427| 11.5%  
> Bartlomiej Zolnierkiewicz| 108056| 7.1%  
> Mauro Carvalho Chehab| 62719| 5.2%  
> Jing Huang| 49189| 3.2%  
> Forest Bond| 45009| 3.0%  
> Ben Hutchings| 37418| 2.5%  
> Eilon Greenstein| 28008| 1.8%  
> Mark Brown| 24516| 1.6%  
> Brian Swetland| 22775| 1.5%  
> Hank Janssen| 19681| 1.3%  
> Leo Chen| 17458| 1.2%  
> Palash Bandyopadhyay| 16790| 1.1%  
> Alan Cox| 16466| 1.1%  
> Mithlesh Thukral| 15173| 1.0%  
> Jerome Glisse| 14343| 0.9%  
> Michael Chan| 13415| 0.9%  
> Martyn Welch| 12480| 0.8%  
> Iliyan Malchev| 12172| 0.8%  
> Jesse Brandeburg| 11051| 0.7%  
  
As has become traditional, Greg Kroah-Hartman and Bartlomiej Zolnierkiewicz feature at the top of both lists. Much of Greg's work had to do with the cleaning up of Microsoft's "hv" drivers. His state of mind during this process is best assessed from the commit messages, which tend to read like [this one](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=ce9ea4cf49b0ba4c67be71550d6cd955b718e967>): 

The Linux kernel doesn't have all caps structures, we don't like to shout at our programmers, it makes them grumpy. Instead, we like to sooth them with small, rounded letters, which puts them in a nice, compliant mood, and makes them more productive and happier, allowing them more fufilling lives overall. 

Greg also removed some drivers from the staging tree, shrinking the kernel by over 100,000 lines. 

The bulk of Bartlomiej's work is also in the staging tree, and that is mostly concerned with fixing up a series of rather unloved wireless network drivers. These patches are somewhat controversial; the wireless developers would rather see that effort going into a different set of non-staging drivers. But those drivers are not yet ready for prime time, and, meanwhile, people are using the staging drivers. Wireless drivers were also the focus of Johannes Berg's work; he has made a long set of improvements to the mac80211 subsystem and its cfg80211 configuration interface. Mark Brown continues to contribute large amounts of code in support of Wolfson Micro's components, and Paul Mundt remains active as the Super-H maintainer. 

In the "lines changed" column, Mauro Carvalho Chehab contributed a lot of patches as the Video4Linux2 maintainer. Jing Huang contributed the Brocade BFA FC SCSI driver, and Forest Bond added the VT6656 wireless driver to the staging tree. 

Developers working on 2.6.32 were supported by (at least) 196 employers. The most active companies this time around are: 

> Most active 2.6.32 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1845| 17.1%  
> Red Hat| 1028| 9.5%  
> (Unknown)| 933| 8.7%  
> Intel| 888| 8.2%  
> Novell| 662| 6.1%  
> IBM| 603| 5.6%  
> Oracle| 319| 3.0%  
> Renesas Technology| 264| 2.5%  
> AMD| 251| 2.3%  
> Nokia| 204| 1.9%  
> Fujitsu| 201| 1.9%  
> Atheros Communications| 197| 1.8%  
> (Consultant)| 195| 1.8%  
> (Academia)| 167| 1.6%  
> Texas Instruments| 155| 1.4%  
> Wolfson Micro| 153| 1.4%  
> Broadcom| 149| 1.4%  
> HP| 130| 1.2%  
> Analog Devices| 124| 1.2%  
> Pengutronix| 119| 1.1%  
> | By lines changed  
> ---  
> (None)| 282017| 18.6%  
> Novell| 256808| 16.9%  
> Red Hat| 150781| 9.9%  
> Broadcom| 84904| 5.6%  
> Intel| 79267| 5.2%  
> (Unknown)| 77122| 5.1%  
> Brocade| 49189| 3.2%  
> Logic Supply| 45165| 3.0%  
> Google| 40936| 2.7%  
> IBM| 29616| 2.0%  
> Wolfson Micro| 25577| 1.7%  
> Texas Instruments| 24824| 1.6%  
> Renesas Technology| 24507| 1.6%  
> Nokia| 24192| 1.6%  
> Microsoft| 19696| 1.3%  
> Oracle| 19410| 1.3%  
> (Consultant)| 18774| 1.2%  
> Conexant| 16790| 1.1%  
> LinSysSoft Technologies| 15173| 1.0%  
> GE Fanuc| 12495| 0.8%  
  
The sharp-eyed reader will notice that Red Hat has fallen below 10% of the total changes - the first time that has happened since [the 2.6.21 development cycle](<http://lwn.net/Articles/224760/>) in early 2007. The number of changes from Red Hat this time around is only slightly lower than the usual, though; what's happening is that some of the other companies are catching up. 

There are a couple of other interesting entries here. Google takes a lot of grief for not contributing back, but that company was the source of a fair amount of code going into 2.6.32. Much of that was support for the HTC "Dream" (aka G1 or ADP1) phone platform, but Google also contributed to control groups, ext4, memory management, IPVS, and libata. And one may have never expected to see Microsoft show up on the list of top kernel contributors, but the hv drivers put it there for 2.6.32. 

The numbers for signoffs have not changed much from previous cycles: 

> Top non-author signoffs in 2.6.32  
> ---  
> | Individuals  
> ---  
> David S. Miller| 996| 10.2%  
> John W. Linville| 994| 10.2%  
> Greg Kroah-Hartman| 788| 8.1%  
> Andrew Morton| 786| 8.1%  
> Ingo Molnar| 501| 5.1%  
> Mauro Carvalho Chehab| 398| 4.1%  
> James Bottomley| 310| 3.2%  
> Len Brown| 188| 1.9%  
> Paul Mundt| 171| 1.8%  
> Russell King| 165| 1.7%  
> | Employers  
> ---  
> Red Hat| 3606| 37.1%  
> Novell| 1309| 13.5%  
> Intel| 906| 9.3%  
> Google| 793| 8.2%  
> (None)| 445| 4.6%  
> IBM| 384| 3.9%  
> (Consultant)| 274| 2.8%  
> Renesas Technology| 180| 1.9%  
> Wolfson Micro| 155| 1.6%  
> Oracle| 138| 1.4%  
  
If anything, the subsystem maintainers are concentrating even more than before. Fully 2/3 of the patches going into the mainline kernel pass through the hands of developers working for just four companies. 

At the [2009 Kernel Summit](<https://lwn.net/Articles/KernelSummit2009/>), the participants concluded that, while improvements can always be made, the process as a whole is working well. The picture that comes from these numbers suggests the same conclusion: the kernel development machine continues to absorb massive numbers of changes from a wide development community while continuing to produce stable, increasingly functional releases.
