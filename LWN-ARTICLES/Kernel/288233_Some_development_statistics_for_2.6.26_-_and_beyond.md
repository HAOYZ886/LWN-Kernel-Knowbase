---
title: "Some development statistics for 2.6.26 - and beyond"
url: https://lwn.net/Articles/288233/
date: "July 2, 2008"
category: "Releases-2.6.26"
author: "By Jonathan Corbet July 2, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 2, 2008

When 2.6.26-rc1 was released, your editor noted that, at a mere 7500 commits, it looked like 2.6.26 would be a smaller than usual development cycle. Interestingly, though, 2.6.26 has caught up. As of this writing (waiting for 2.6.26-rc9), this development cycle has incorporated 10,102 changesets for a net addition of 169,439 lines of code to the kernel. That makes it still significantly smaller than 2.6.25, but it is, by no means small. The developer base remains as broad as ever: 1065 developers (representing some 150 companies) have contributed to 2.6.26; just over 1/3 of those developers contributed one single changeset. 

The 2.6 development model says that the bulk of the changes should be merged during the merge window (before the -rc1 release), with only fixes coming thereafter. Here's how things break down for recent releases: 

> Release| Changesets merged  
> ---|---  
> For -rc1| after -rc1  
> 2.6.23| 4505| 2570  
> 2.6.24| 7132| 3221  
> 2.6.25| 9629| 3078  
> 2.6.26| 7555| 2577  
  
So, while the bulk of the big patches enter the kernel during the merge window, at least 25% of the total - and often more - come thereafter. That's a lot of fixes. 

So who were the most active developers this time around? Here's the top 20: 

> Most active 2.6.26 developers  
> ---  
> | By changesets  
> ---  
> Harvey Harrison| 218| 2.2%  
> Bartlomiej Zolnierkiewicz| 197| 1.9%  
> Glauber Costa| 195| 1.9%  
> Adrian Bunk| 180| 1.8%  
> Joe Perches| 160| 1.6%  
> Pavel Emelyanov| 148| 1.5%  
> Ingo Molnar| 144| 1.4%  
> Denis V. Lunev| 140| 1.4%  
> Michael Krufky| 130| 1.3%  
> Mauro Carvalho Chehab| 116| 1.1%  
> Al Viro| 114| 1.1%  
> David S. Miller| 103| 1.0%  
> Tejun Heo| 96| 0.9%  
> Johannes Berg| 96| 0.9%  
> Alan Cox| 91| 0.9%  
> Takashi Iwai| 88| 0.9%  
> YOSHIFUJI Hideaki| 85| 0.8%  
> Alexey Starikovskiy| 84| 0.8%  
> Ivo van Doorn| 80| 0.8%  
> Bjorn Helgaas| 77| 0.8%  
> | By changed lines  
> ---  
> Stephen Hemminger| 41762| 5.9%  
> Adrian Bunk| 28523| 4.0%  
> David S. Miller| 19178| 2.7%  
> Steven Toth| 18681| 2.6%  
> Ben Hutchings| 15535| 2.2%  
> Frank Blaschka| 14527| 2.0%  
> Xiantao Zhang| 12935| 1.8%  
> Hans Verkuil| 12393| 1.7%  
> Tejun Heo| 10462| 1.5%  
> Sebastian Siewior| 9519| 1.3%  
> Harvey Harrison| 9161| 1.3%  
> Peter Tiedemann| 8483| 1.2%  
> Matthew Wilcox| 8059| 1.1%  
> Paul Walmsley| 7635| 1.1%  
> Kumar Gala| 7152| 1.0%  
> Andrew Victor| 7062| 1.0%  
> Johannes Berg| 6544| 0.9%  
> Glauber Costa| 6260| 0.9%  
> Mike Frysinger| 6177| 0.9%  
> Joe Perches| 5773| 0.8%  
  
In terms of the number of changesets merged, Harvey Harrison got to the top of the list with a wide variety of of janitorial fixes. Bartlomiej Zolnierkiewicz continues to put significant effort into cleaning up the IDE subsystem, even though most distributors have moved away from that code and are using the newer PATA layer instead. Glauber Costa has been tirelessly working in the x86 architecture code; in particular, he continues to work toward the goal of unifying the 32-bit and 64-bit code to the greatest extent possible. Adrian Bunk has made a career of cleaning up the code base and eliminating unneeded code. And Joe Perches dedicated much time to eliminating warnings from the checkpatch.pl script. 

There have been complaints from the developers that the volume of "cleanup" patches is reaching a point that it is drowning out the rest and interfering with "real work." We're seeing some of that volume here, with three of the top five changeset contributors doing cleanup work - some of which is seen to be more valuable than the rest. 

On the lines changed side, we see a mostly different set of developers. In this case, the top slots were earned by deleting code. Stephen Hemminger finally succeeded in getting rid of the old sk98lin driver. Adrian Bunk tore out the bcm43xx driver, the ieee80311 software MAC layer, the xircom_tulip_cb driver, and various other bits and pieces. David Miller removed a bunch of old SPARC code, but replaced it with various other facilities; he also took the PowerPC low-level memory manager and made it generic. Steven Toth works in the Video4Linux layer; he added some new drivers and a bunch of cleanups. Ben Hutchings added the Solarstorm SFC4000 driver. 

When one thinks about 2.6.26 features, the things that come to mind include KGDB, almost-ready network namespaces, almost-ready mesh networking support, a working (shall we say "almost ready"?) realtime group scheduler, read-only bind mounts, page attribute table support, the object debugging infrastructure, and, of course, the vast pile of new drivers. One has to look hard to find the developers behind that work in the lists above (some of them are certainly there). Which just reinforces an important point: there is interest and information in counting changesets and lines changed, but the correlation between those numbers and serious accomplishments in kernel programming is weak at best. Unfortunately, "real work" is awfully hard to measure in any sort of automated way. 

So what the heck; we'll go back to the numbers we can measure. Here's the most active companies for 2.6.26: 

> Most active 2.6.26 employers  
> ---  
> | By changesets  
> ---  
> (None)| 2085| 20.6%  
> Red Hat| 1130| 11.2%  
> (Unknown)| 906| 8.9%  
> IBM| 609| 6.0%  
> Novell| 597| 5.9%  
> Intel| 469| 4.6%  
> Parallels| 312| 3.1%  
> SGI| 211| 2.1%  
> Movial| 180| 1.8%  
> Oracle| 142| 1.4%  
> Analog Devices| 134| 1.3%  
> HP| 124| 1.2%  
> MontaVista| 122| 1.2%  
> (Consultant)| 116| 1.1%  
> Freescale| 109| 1.1%  
> QLogic| 97| 1.0%  
> Fujitsu| 95| 0.9%  
> Google| 94| 0.9%  
> (Academia)| 89| 0.9%  
> Marvell| 88| 0.9%  
> | By lines changed  
> ---  
> (None)| 111703| 15.7%  
> IBM| 73601| 10.3%  
> Red Hat| 56331| 7.9%  
> Intel| 50297| 7.1%  
> (Unknown)| 44699| 6.3%  
> Vyatta| 41835| 5.9%  
> Novell| 33745| 4.7%  
> Movial| 28632| 4.0%  
> Hauppauge| 20234| 2.8%  
> Analog Devices| 18363| 2.6%  
> (Consultant)| 16397| 2.3%  
> Solarflare | 15585| 2.2%  
> Freescale| 15090| 2.1%  
> MontaVista| 14013| 2.0%  
> QLogic| 13327| 1.9%  
> SGI| 10351| 1.5%  
> Marvell| 7881| 1.1%  
> Wind River| 7770| 1.1%  
> Oracle| 7680| 1.1%  
> Pengutronix| 7334| 1.0%  
  
This list tends not to change too much from one release to the next; in particular, the top companies are always the same. 

If we look at who is attaching Signed-off-by tags to code they didn't write, we get a sense for who the gatekeepers to the kernel are. These are the developers and companies who are herding code into the mainline: 

> Sign-offs in the 2.6.26 kernel  
> ---  
> | By developer  
> ---  
> Andrew Morton| 1377| 14.1%  
> Ingo Molnar| 961| 9.8%  
> David S. Miller| 667| 6.8%  
> John W. Linville| 551| 5.6%  
> Mauro Carvalho Chehab| 543| 5.6%  
> Jeff Garzik| 471| 4.8%  
> Thomas Gleixner| 279| 2.9%  
> Greg KH| 267| 2.7%  
> Linus Torvalds| 256| 2.6%  
> Paul Mackerras| 220| 2.2%  
> Takashi Iwai| 208| 2.1%  
> James Bottomley| 203| 2.1%  
> Len Brown| 200| 2.0%  
> Russell King| 167| 1.7%  
> Avi Kivity| 160| 1.6%  
> Bryan Wu| 140| 1.4%  
> Roland Dreier| 130| 1.3%  
> Lachlan McIlroy| 108| 1.1%  
> Bartlomiej Zolnierkiewicz| 94| 1.0%  
> Ralf Baechle| 93| 1.0%  
> | By employer  
> ---  
> Red Hat| 3010| 30.8%  
> Google| 1378| 14.1%  
> (None)| 1000| 10.2%  
> Novell| 731| 7.5%  
> IBM| 577| 5.9%  
> Intel| 497| 5.1%  
> linutronix| 283| 2.9%  
> Linux Foundation| 256| 2.6%  
> (Unknown)| 206| 2.1%  
> (Consultant)| 206| 2.1%  
> Hansen Partnership| 203| 2.1%  
> SGI| 166| 1.7%  
> Qumranet| 160| 1.6%  
> Analog Devices| 149| 1.5%  
> Cisco| 130| 1.3%  
> MIPS Technologies| 93| 1.0%  
> Oracle| 57| 0.6%  
> Freescale| 55| 0.6%  
> Renesas Technology| 54| 0.6%  
> Univ. of Michigan CITI| 47| 0.5%  
  
Once again, these numbers tend not to change that much from one development cycle to the next. Subsystem maintainers do not change often. 

### What's next?

This is the first full development cycle where the linux-next tree was in operation. At this stage in the cycle, linux-next should look very much like 2.6.27 - or, at least, 2.6.27-rc1. Your editor pulled the July 2 linux-next tree and ran some statistics; this tree contains 6527 changesets from 619 developers. Just over 400,000 lines of code are touched, with a net addition of 38,000 lines. 

If linux-next is to be believed, the most active 2.6.27 developers will be: 

> Most active pre-2.6.27 developers  
> ---  
> | By changesets  
> ---  
> Avi Kivity| 499| 7.6%  
> Artem Bityutskiy| 292| 4.5%  
> Bartlomiej Zolnierkiewicz| 150| 2.3%  
> Ingo Molnar| 142| 2.2%  
> Yinghai Lu| 139| 2.1%  
> Adrian Hunter| 121| 1.9%  
> Alan Cox| 101| 1.5%  
> Xiantao Zhang| 100| 1.5%  
> Tomas Winkler| 91| 1.4%  
> Rusty Russell| 89| 1.4%  
> David Woodhouse| 86| 1.3%  
> Adrian Bunk| 84| 1.3%  
> Steven Rostedt| 83| 1.3%  
> Jonathan Corbet| 74| 1.1%  
> Arnd Bergmann| 73| 1.1%  
> Jean Delvare| 67| 1.0%  
> Harvey Harrison| 64| 1.0%  
> David Chinner| 63| 1.0%  
> Lennert Buytenhek| 61| 0.9%  
> Thomas Gleixner| 61| 0.9%  
> | By changed lines  
> ---  
> David Woodhouse| 44833| 6.7%  
> Artem Bityutskiy| 41891| 6.3%  
> Eilon Greenstein| 18614| 2.8%  
> Xiantao Zhang| 17223| 2.6%  
> Alan Cox| 14850| 2.2%  
> Jaswinder Singh| 10805| 1.6%  
> David Brownell| 9618| 1.4%  
> Stephen Rothwell| 9043| 1.4%  
> Lennert Buytenhek| 9029| 1.3%  
> Avi Kivity| 8593| 1.3%  
> Steven Rostedt| 7923| 1.2%  
> Adrian Bunk| 7424| 1.1%  
> Laurent Pinchart| 7200| 1.1%  
> Yinghai Lu| 6850| 1.0%  
> Yaniv Rosner| 6512| 1.0%  
> Carsten Otte| 6442| 1.0%  
> Tomas Winkler| 6250| 0.9%  
> Josh Boyer| 5292| 0.8%  
> Adrian Hunter| 5155| 0.8%  
> Michael Chan| 5133| 0.8%  
  
These numbers reflect a number of the larger developments which can be expected for 2.6.27: incredible amounts of KVM work, the merging of the UBIFS filesystem, the ftrace tracing framework, a lot of reworking of the TTY layer, a lot of firmware thrashing, and ongoing big kernel lock removal work. 

It will be most interesting to see how these numbers compare with what actually shows up in 2.6.27-rc1. Recent numbers suggest that quite a few patches will hit the mainline without having been in the linux-next tree - either that, or 2.6.27 will be a relatively small release. If nothing else, we will see which developers do not yet get their work into linux-next for integration testing ahead of the merge window.
