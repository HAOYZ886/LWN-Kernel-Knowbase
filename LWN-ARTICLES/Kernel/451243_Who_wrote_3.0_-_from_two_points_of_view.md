---
title: "Who wrote 3.0 - from two points of view"
url: https://lwn.net/Articles/451243/
date: "July 13, 2011"
category: "Releases-3.0"
author: "By Jonathan Corbet July 13, 2011"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
July 13, 2011

Linus Torvalds had hoped to release the 3.0 kernel after -rc6, but reality, as is its wont, intervened; thus, [3.0-rc7](<https://lwn.net/Articles/451245/>) was released on July 11\. That probably is the last development release for 3.0, though. Tradition dictates that we take a look at the contributor statistics for this development cycle, which we will now do. 

This kernel release inaugurates the beginning of the 3.x series of kernels. As has been mentioned many times here, there is nothing particularly special about the 3.0 release; it has been, in many ways, a relatively boring development cycle. But it still provides a good opportunity to look back over a longer period of time. But, before doing that, we'll start with this cycle, which has, as of 3.0-rc7, seen 9,007 changesets contributed by 1,110 developers. The kernel grew 113,000 lines in this development cycle - a relatively modest figure by contemporary standards. 

The most active developers during this cycle were: 

> Most active 3.0 developers  
> ---  
> | By changesets  
> ---  
> K. Y. Srinivasan| 343| 3.8%  
> David S. Miller| 176| 2.0%  
> Dan Williams| 149| 1.7%  
> Jonathan Cameron| 119| 1.3%  
> Takashi Iwai| 108| 1.2%  
> Mark Brown| 91| 1.0%  
> Johannes Berg| 84| 0.9%  
> Peter Zijlstra| 80| 0.9%  
> Sage Weil| 79| 0.9%  
> Tejun Heo| 78| 0.9%  
> Joe Perches| 77| 0.9%  
> Michał Mirosław| 77| 0.9%  
> Konrad Rzeszutek Wilk| 76| 0.8%  
> Jamie Iles| 75| 0.8%  
> Alex Deucher| 71| 0.8%  
> Artem Bityutskiy| 69| 0.8%  
> Steven Rostedt| 66| 0.7%  
> Mike Frysinger| 63| 0.7%  
> Sujith Manoharan| 62| 0.7%  
> Avi Kivity| 58| 0.6%  
> | By changed lines  
> ---  
> Dan Williams| 82466| 9.1%  
> Larry Finger| 74643| 8.3%  
> Dmitry Kravkov| 38960| 4.3%  
> Vasanthakumar Thiagarajan| 33618| 3.7%  
> Mauro Carvalho Chehab| 26815| 3.0%  
> Bing Zhao| 25576| 2.8%  
> Ralph Metzler| 19933| 2.2%  
> Takahiro Hirofuchi| 19318| 2.1%  
> Chaoming Li| 14743| 1.6%  
> Jonathan Cameron| 14574| 1.6%  
> Chris Metcalf| 12144| 1.3%  
> Luis R. Rodriguez| 11443| 1.3%  
> Dave Jiang| 11006| 1.2%  
> Wolfram Sang| 9886| 1.1%  
> K. Y. Srinivasan| 9709| 1.1%  
> Mark Brown| 9127| 1.0%  
> Arend van Spriel| 7667| 0.8%  
> Kenji Toyama| 7528| 0.8%  
> Alan Cox| 7449| 0.8%  
> Takashi Iwai| 7410| 0.8%  
  
K. Y. Srinivasan topped the list of changeset contributors with a massive set of cleanups to the Microsoft HV driver in the staging tree; it's impressive to see how much cleanup less than 15,000 lines of code can require. David Miller made a lot of changes in the networking subsystem; some were warning fixes and such, while others were more substantial. Dan Williams contributed Intel's "isci" storage driver, merged in 3.0-rc6. Jonathan Cameron contributed a lot of work to rationalize the industrial I/O (iio) subsystem and prepare it for an eventual merge into the mainline. Takashi Iwai continues to do large amounts of work in the ALSA sound driver subsystem. 

The isci driver put Dan Williams at the top of the "lines changed" column. Larry Finger's contribution is largely negative (in line counts - not in value): he removed the rt2860sta and rt2870sta drivers from the staging tree now that the mainline driver can replace them. Dmitry Kravkov appears due to a firmware update; the bnx2x driver is one of the few which still has firmware in the mainline kernel tree. Vasanthakumar Thiagarajan also removed a lot of code, mostly through the process of eliminating duplication between Atheros wireless drivers. Mauro Carvalho Chehab removed the obsolete Micronas drx397xD driver. 

A total of 184 employers (that we were able to identify) participated in the 3.0 cycle; the most active among them were: 

> Most active 3.0 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1085| 12.0%  
> Red Hat| 1000| 11.1%  
> Intel| 839| 9.3%  
> (Unknown)| 569| 6.3%  
> Novell| 441| 4.9%  
> IBM| 374| 4.2%  
> Microsoft| 361| 4.0%  
> Atheros Communications| 241| 2.7%  
> Texas Instruments| 234| 2.6%  
> Broadcom| 222| 2.5%  
> Oracle| 187| 2.1%  
> AMD| 162| 1.8%  
> Nokia| 158| 1.8%  
> Fujitsu| 154| 1.7%  
> Google| 129| 1.4%  
> University of Cambridge| 119| 1.3%  
> Analog Devices| 118| 1.3%  
> (Consultant)| 113| 1.3%  
> Samsung| 103| 1.1%  
> Wolfson Microelectronics| 103| 1.1%  
> | By lines changed  
> ---  
> Intel| 163232| 18.1%  
> (None)| 152840| 16.9%  
> Broadcom| 61948| 6.9%  
> Red Hat| 59079| 6.5%  
> Atheros Communications| 53268| 5.9%  
> Marvell| 31118| 3.4%  
> (Unknown)| 29261| 3.2%  
> IBM| 20587| 2.3%  
> Metzler Brothers Systementwicklung GbR| 19933| 2.2%  
> Novell| 19578| 2.2%  
> University of Cambridge| 16969| 1.9%  
> Pengutronix| 16207| 1.8%  
> Realsil Microelectronics| 14876| 1.6%  
> Analog Devices| 12998| 1.4%  
> Tilera| 12257| 1.4%  
> Freescale| 11637| 1.3%  
> Microsoft| 11564| 1.3%  
> Texas Instruments| 10802| 1.2%  
> Wolfson Microelectronics| 10051| 1.1%  
> Samsung| 9784| 1.1%  
  
There are few surprises here. Microsoft at 4% of the total changes is unusual; one assumes that presence will not be permanent: even the HV drivers can only need so much cleaning up. The percentage of changes from hobbyists continues to drop; whether that's a bad thing (the kernel is becoming increasingly unapproachable to volunteer developers) or a good thing (it's impossible for anybody who can hack the kernel to remain unemployed) is still not clear. 

#### A longer-term look

The release of 3.0 provides as good an opportunity as any to look at the entire 2.6 series. Thanks to the [BitKeeper history tree](<http://git.kernel.org/?p=linux/kernel/git/tglx/history.git;a=summary>) put together by Thomas Gleixner, it is possible to get detailed information back almost to the beginning of the 2.5 development cycle, which can be thought of as the set of -rc kernels leading up to 2.6.0. This information is far from complete, unfortunately. The 2.5.0 through 2.5.3 releases predate the BitKeeper transition, and thus appear as big patches from Linus. Even thereafter, a lot of early changes appear to have been contributed by the maintainer they passed through instead of the actual author; it took a while to establish the infrastructure to properly credit all work. Still, there is enough data there to work with. 

The history from the beginning of the 2.5 development series covers about 9.5 years of development. During this time, some 291,664 changesets were contributed by 8,078 developers; those changes added 10.5 million lines of code. Here are the most active developers over that extended period: 

> Most active developers since 2.5.0  
> ---  
> | By changesets  
> ---  
> Andrew Morton| 7638| 2.6%  
> David S. Miller| 5203| 1.8%  
> Al Viro| 3828| 1.3%  
> Greg Kroah-Hartman| 3309| 1.1%  
> Russell King| 3226| 1.1%  
> Alan Cox| 2609| 0.9%  
> Ingo Molnar| 2599| 0.9%  
> Stephen Hemminger| 2535| 0.9%  
> Bartlomiej Zolnierkiewicz| 2485| 0.9%  
> Linus Torvalds| 2479| 0.8%  
> Christoph Hellwig| 2429| 0.8%  
> Takashi Iwai| 2414| 0.8%  
> Adrian Bunk| 2306| 0.8%  
> Tejun Heo| 2205| 0.8%  
> Thomas Gleixner| 2205| 0.8%  
> Paul Mundt| 2113| 0.7%  
> Dave Jones| 2067| 0.7%  
> Randy Dunlap| 1853| 0.6%  
> Ralf Baechle| 1786| 0.6%  
> Johannes Berg| 1770| 0.6%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 738134| 2.3%  
> Bartlomiej Zolnierkiewicz| 553077| 1.7%  
> Andrew Morton| 537737| 1.7%  
> Alan Cox| 432023| 1.4%  
> Jaroslav Kysela| 387649| 1.2%  
> Adrian Bunk| 380691| 1.2%  
> James Bottomley| 367435| 1.2%  
> Linus Torvalds| 325954| 1.0%  
> Ralf Baechle| 319859| 1.0%  
> Paul Mackerras| 279454| 0.9%  
> Sam Ravnborg| 270118| 0.8%  
> David S. Miller| 254574| 0.8%  
> Christoph Hellwig| 238749| 0.8%  
> Mauro Carvalho Chehab| 232793| 0.7%  
> Uwe Kleine-Konig| 215560| 0.7%  
> Russell King| 209362| 0.7%  
> Benjamin Herrenschmidt| 195707| 0.6%  
> Jeff Garzik| 190724| 0.6%  
> Paul Mundt| 185781| 0.6%  
> David Howells| 183872| 0.6%  
  
It should be repeated that these numbers are highly approximate. For example, while Andrew Morton was indeed a prolific code contributor in the 2.5.x and early 2.6 days, he didn't write quite _that_ many patches; a lot of patches from others that went through him lost their authorship information on the way. That information is generally present in the changelog - somebody could try to make a new repository with proper credits given some time - but, for now, we'll have to make do with fuzzy numbers. The per-employer numbers are necessarily even fuzzier - to the point that they are most likely not worth showing here. Suffice to say that, in general form, they resemble the numbers we have been showing for the last few years. 

For those who are curious about just the post-2.6.0 kernels, the numbers don't change that much. Since 2.6.0, there have been 264,706 changesets contributed by 7,725 developers adding 8.7 million lines of code. 

One other exercise with this data seemed interesting: a determination of who have been the most consistent contributors over those nine years and some. After running a script to track which developers contributed to each major release, twelve developers were found who had contributed to all 41 of them. Additionally, a handful of developers have gotten code into _almost_ every release. The most consistent developers are: 

> Most consistent developers 2.6.0-3.0  
> ---  
> Developer| Releases| Missed releases  
> Linus Torvalds| 41|   
> David S. Miller| 41|   
> Greg Kroah-Hartman| 41|   
> Andrew Morton| 41|   
> Christoph Hellwig| 41|   
> Alan Stern| 41|   
> James Bottomley| 41|   
> Randy Dunlap| 41|   
> Russell King| 41|   
> Al Viro| 41|   
> Stephen Hemminger| 41|   
> Andi Kleen| 41|   
> Jens Axboe| 40| v2.6.1  
> Jean Delvare| 40| v2.6.4  
> Dave Jones| 40| v2.6.35  
> Benjamin Herrenschmidt| 40| v2.6.1  
> Jeff Garzik| 40| v2.6.36  
> Ingo Molnar| 39| v2.6.2 v2.6.5  
> Herbert Xu| 39| v2.6.3 v2.6.5  
> Patrick McHardy| 39| v2.6.2 v2.6.6  
> Dmitry Torokhov| 38| v2.6.3 v2.6.4 v2.6.6  
> Rusty Russell| 38| v2.6.1 v2.6.15 v2.6.39  
> Matthew Wilcox| 38| v2.6.14 v2.6.36 v3.0  
> Dave Kleikamp| 38| v2.6.26 v2.6.33 v2.6.37  
> Len Brown| 38| v2.6.1 v2.6.17 v2.6.39  
> Oliver Neukum| 38| v2.6.4 v2.6.14 v2.6.37  
> Wim Van Sebroeck| 38| v2.6.4 v2.6.6 v3.0  
> Andrew Vasquez| 38| v2.6.0 v2.6.1 v2.6.5  
> James Morris| 38| v2.6.16 v2.6.37 v2.6.39  
> Neil Brown| 37| v2.6.1 v2.6.2 v2.6.3 v2.6.6  
> Trond Myklebust| 37| v2.6.1 v2.6.2 v2.6.8 v2.6.10  
> Paul Mackerras| 37| v2.6.1 v2.6.3 v2.6.38 v2.6.39  
> Bjorn Helgaas| 37| v2.6.3 v2.6.20 v2.6.39 v3.0  
> Tony Lindgren| 37| v2.6.0 v2.6.1 v2.6.5 v2.6.20  
> Nicolas Pitre| 37| v2.6.3 v2.6.4 v2.6.5 v2.6.23  
> Stephen Rothwell| 37| v2.6.1 v2.6.2 v2.6.3 v2.6.7  
> David Howells| 36| v2.6.1 v2.6.2 v2.6.3 v2.6.4 v2.6.6  
> Eric Sandeen| 36| v2.6.1 v2.6.8 v2.6.11 v2.6.17 v3.0  
> Ralf Baechle| 36| v2.6.1 v2.6.3 v2.6.4 v2.6.7 v2.6.38  
> Arjan van de Ven| 36| v2.6.1 v2.6.3 v2.6.13 v2.6.14 v3.0  
> David Brownell| 36| v2.6.33 v2.6.34 v2.6.37 v2.6.38 v3.0  
  
Your editor, who only got changes into 32 releases during this time, knows what an accomplishment it is to consistently contribute to every release over such a long period of time. 

But, then, creating the kernel and the development process we have over the course of the last 20 years is an impressive accomplishment. There are few development projects which have lasted this long, gone this far, and have been more vital than ever. It has been fun to watch. It seems likely that things will remain just as fun over the next 20 years - one could argue that we have just begun.
