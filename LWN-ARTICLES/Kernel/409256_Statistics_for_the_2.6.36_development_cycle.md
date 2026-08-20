---
title: Statistics for the 2.6.36 development cycle
url: https://lwn.net/Articles/409256/
date: "October 13, 2010"
category: "Releases-2.6.36"
author: "By Jonathan Corbet October 13, 2010"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 13, 2010

As this is being written, the last 2.6.36 prepatch has (with luck) been released and the final release can be expected within a few days. So it is time to have a look at how this development cycle has gone. There are a couple of things which distinguish 2.6.36 from its predecessors in interesting ways. 

The 2.6.36 kernel will incorporate about 9400 changesets contributed by 1159 developers. It thus continues a recent trend toward less active development cycles; here is what we have seen over the course of the last year or so: 

> Release| Changes  
> ---|---  
> 2.6.31| 10,883  
> 2.6.32| 10,998  
> 2.6.33| 10,871  
> 2.6.34| 9,443  
> 2.6.35| 9,801  
> 2.6.36| ~9,400  
  
The work which pushed up the changeset numbers in previous development cycles (shoveling out-of-tree code into the staging directory being at the top of the list) continues to wind down, as does work in other areas (like new filesystems). As a result, the kernel is going through a period of relatively low flux - but only relative to the last couple of years - and stabilization. That said, it's worth noting that, unless something unexpected happens, the 2.6.36 development cycle will be one of the shortest in recent memory; as a result, the number of changesets merged per day is the highest since 2.6.30. 

Perhaps more interesting is this set of numbers: in 2.6.36, the development community added 604,000 lines of code and deleted 651,000 - for a total loss of almost 47,000 lines of code. This is the first time since the beginning of the git era that the size of the kernel source has gone down. Given that, perhaps it is appropriate to start with a look at who has been so busily removing code from the kernel: 

> Most lines removed - 2.6.36  
> ---  
> Sam Ravnborg| 205813| 31.6%  
> Benjamin Herrenschmidt| 133666| 20.5%  
> Amerigo Wang| 19145| 2.9%  
> Tony Luck| 8418| 1.3%  
> Greg Kroah-Hartman| 7094| 1.1%  
> Kiran Divekar| 4487| 0.7%  
> Palash Bandyopadhyay| 4457| 0.7%  
> Vincent Sanders| 3467| 0.5%  
> Dave Jones| 2600| 0.4%  
> Christoph Hellwig| 2163| 0.3%  
  
Sam Ravnborg and Ben Herrenschmidt both got to the top of the list through the removal of a bunch of defconfig files, part of a general cleanup inspired by [some grumpiness from Linus](<http://lwn.net/Articles/391372/>) back in June; Sam also finished up some SPARC unification work. Amerigo Wang removed a number of old and unused drivers. Between the three of them, they got rid of almost 360,000 lines of code - a laudable bit of work. 

Looking at code changes in general for the 2.6.36 development cycle yields this picture: 

> Most active 2.6.36 developers  
> ---  
> | By changesets  
> ---  
> Vasiliy Kulikov| 160| 1.7%  
> Eric Paris| 124| 1.3%  
> Dan Carpenter| 122| 1.3%  
> Chris Wilson| 117| 1.3%  
> Eric Dumazet| 108| 1.2%  
> Uwe Kleine-Konig| 103| 1.1%  
> Axel Lin| 98| 1.0%  
> Johannes Berg| 97| 1.0%  
> Al Viro| 96| 1.0%  
> Julia Lawall| 89| 1.0%  
> Tejun Heo| 88| 0.9%  
> Joe Perches| 83| 0.9%  
> Christoph Hellwig| 73| 0.8%  
> Alex Deucher| 71| 0.8%  
> Ben Skeggs| 69| 0.7%  
> John W. Linville| 68| 0.7%  
> Stefan Richter| 64| 0.7%  
> Stephen M. Cameron| 62| 0.7%  
> Felix Fietkau| 60| 0.6%  
> Randy Dunlap| 59| 0.6%  
> | By changed lines  
> ---  
> Sam Ravnborg| 208270| 19.4%  
> Benjamin Herrenschmidt| 134811| 12.5%  
> Chris Metcalf| 53204| 4.9%  
> Omar Ramirez Luna| 51087| 4.8%  
> Amerigo Wang| 19191| 1.8%  
> Jarod Wilson| 16020| 1.5%  
> Felix Fietkau| 11898| 1.1%  
> Alan Olsen| 11650| 1.1%  
> Mike Thomas| 11087| 1.0%  
> Lars-Peter Clausen| 10795| 1.0%  
> Tony Luck| 9351| 0.9%  
> Tetsuo Handa| 7955| 0.7%  
> Casey Leedom| 7888| 0.7%  
> John Johansen| 7591| 0.7%  
> Greg Kroah-Hartman| 7195| 0.7%  
> Charles Clement| 6864| 0.6%  
> Dmitry Kravkov| 6754| 0.6%  
> Kiran Divekar| 6753| 0.6%  
> Ben Collins| 6540| 0.6%  
> Christoph Hellwig| 6045| 0.6%  
  
On the changesets side, Vasiliy Kulikov leads with a long list of mostly small fixes, mostly in device driver code. The bulk of Eric Paris's work is the addition of the fanotify subsystem - work which, as of this writing, will not be enabled for the 2.6.36 release due to user-space ABI concerns. Dan Carpenter is another master of small fixes, usually for problems identified by static analysis tools. Chris Wilson had a large number of changes to the Intel i915 driver - and seemingly an even larger number fixing the resulting problems. Eric Dumazet's changes were a large number of fixes and improvements to the networking subsystem. 

Three of the top five in the "lines changed" column have already been mentioned above. The other two are Chris Metcalf, who added the new "Tile" architecture, and Omar Ramirez Luna, who added the TI dspbridge driver to the staging tree. 

Only one top-five developer (Dan Carpenter) was also in the top five for 2.6.35; there are a lot of new faces on the list this time around. 

There were 184 employers (that we could identify) who contributed code to the 2.6.36 kernel. The top corporate supporters were: 

> Most active 2.6.36 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1524| 16.3%  
> Red Hat| 1129| 12.1%  
> (Unknown)| 865| 9.2%  
> Intel| 691| 7.4%  
> Novell| 404| 4.3%  
> IBM| 374| 4.0%  
> Nokia| 212| 2.3%  
> Texas Instruments| 189| 2.0%  
> (Academia)| 178| 1.9%  
> Samsung| 178| 1.9%  
> Fujitsu| 160| 1.7%  
> NTT| 151| 1.6%  
> Pengutronix| 145| 1.6%  
> AMD| 131| 1.4%  
> Google| 125| 1.3%  
> (Consultant)| 109| 1.2%  
> Societe Francaise de Radiotelephone| 108| 1.2%  
> QLogic| 107| 1.1%  
> Atheros Communications| 99| 1.1%  
> MiTAC| 98| 1.0%  
> | By lines changed  
> ---  
> (None)| 299115| 27.8%  
> IBM| 151386| 14.1%  
> Red Hat| 76455| 7.1%  
> (Unknown)| 71662| 6.7%  
> Tilera| 64825| 6.0%  
> Texas Instruments| 63521| 5.9%  
> Intel| 55167| 5.1%  
> Novell| 25798| 2.4%  
> Samsung| 14619| 1.4%  
> NTT| 12187| 1.1%  
> Marvell| 10769| 1.0%  
> Chelsio| 10560| 1.0%  
> (Academia)| 10345| 1.0%  
> QLogic| 9873| 0.9%  
> Google| 9503| 0.9%  
> Broadcom| 8391| 0.8%  
> ST Ericsson| 8390| 0.8%  
> Canonical| 8354| 0.8%  
> Nokia| 8060| 0.7%  
> Atheros Communications| 7762| 0.7%  
  
For the most part, this list looks the way it has for most development cycles, but there are a couple of new names here. One is Tilera, the company behind the Tile architecture, which got its support merged for 2.6.36. The other name appearing here for the first time is Canonical, which got the AppArmor security module code merged at last. Meanwhile, one should not forget the other 164 companies which do not appear on the above list; the commercial ecosystem around the Linux kernel remains strong and diverse. 

Finally, your editor decided to rerun [an old experiment](<http://lwn.net/Articles/374574/>) to look at the longevity of code in the kernel. Every line in the kernel source was mapped back to the kernel release where it was last changed, then the totals for each release were plotted. The resulting picture looks like this: 

> ![\[Bar chart\]](https://static.lwn.net/images/ns/kernel/linetags-36.png)

At 1.6% of the total, 2.6.36 represents a relatively small piece of the total code base - the smallest for a long time. Almost 29% of the kernel code still dates back to the beginning of the git era, down from 31% last February. While much of our kernel code is quite new - 31% of the code comes from 2.6.30 or newer - much of it has also hung around for a long time. 

All told, 2.6.36 was a relaxed development cycle with relatively few big new features and a fair amount of cleanup. That is certainly part of how it was able to be stabilized in a shorter-than-usual period, and with fewer than the usual number of regressions (56 [reported](<https://lwn.net/Articles/409419/>) as of October 10, as opposed to 100 for 2.6.35-rc6). Whether 2.6.36 represents a new norm for a slightly slower kernel development process remains to be seen. As of this writing, the linux-next tree contains 5850 changesets, most of which are presumably intended for 2.6.37. Quite a few changes still typically do not appear in linux-next prior to the opening of the merge window, so we should see more changes than that merged for 2.6.37. Still, current linux-next does not look like a huge wave of pent-up changes waiting to fly into the mainline; 2.6.37 may or may not exceed 2.6.36 in the number of changes, but it does not look like it will be breaking any records.
