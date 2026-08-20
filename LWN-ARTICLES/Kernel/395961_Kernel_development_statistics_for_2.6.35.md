---
title: Kernel development statistics for 2.6.35
url: https://lwn.net/Articles/395961/
date: "July 14, 2010"
category: "Releases-2.6.35"
author: "July 14, 2010 This article was contributed by Greg Kroah-Hartman ."
---

July 14, 2010

This article was contributed by [Greg Kroah-Hartman](<http://www.kroah.com/linux/>).

In the tradition of summarizing the statistics of the Linux kernel releases before the actual release of the kernel version itself, here is a summary of what has happened in the Linux kernel tree over the past few months. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

This kernel release has seen 9460 changesets from about 1145 different developers so far. This continues the trend over the past few kernel releases for the size of both the changes as well as the development community as can be seen in this table: 

> Kernel| Patches| Devs  
> ---|---|---  
> 2.6.29 | 11,600 | 1170  
> 2.6.30 | 11,700 | 1130  
> 2.6.31 | 10,600 | 1150  
> 2.6.32 | 10,800 | 1230  
> 2.6.33 | 10,500 | 1150  
> 2.6.34 | 9,100 | 1110  
> 2.6.35 | 9,460 | 1145  
  
Perhaps our years of increasing developer activity -- getting more developers per release and more changes per release -- has finally reached a plateau. If so, that is not a bad thing, as a number of us were wondering what the limits of our community were going to be. At around 10 thousand changes per release, that limit is indeed quite high, so there is no need to be concerned, as the Linux kernel is still, by far, the most active software development project the world has ever seen. 

In looking at the individual developers, the quantity and size of contributions is still quite large: 

> Most active 2.6.35 developers  
> ---  
> | By changesets  
> ---  
> Mauro Carvalho Chehab| 228| 2.3%  
> Dan Carpenter| 135| 1.3%  
> Greg Kroah-Hartman| 134| 1.3%  
> Arnaldo Carvalho de Melo| 121| 1.2%  
> Johannes Berg| 105| 1.0%  
> Ben Dooks| 98| 1.0%  
> Julia Lawall| 96| 1.0%  
> Hans Verkuil| 92| 0.9%  
> Alexander Graf| 84| 0.8%  
> Eric Dumazet| 82| 0.8%  
> Peter Zijlstra| 79| 0.8%  
> Paul Mundt| 79| 0.8%  
> Johan Hovold| 75| 0.7%  
> Tejun Heo| 74| 0.7%  
> Stephen Hemminger| 74| 0.7%  
> Mark Brown| 71| 0.7%  
> Sage Weil| 70| 0.7%  
> Alex Deucher| 68| 0.7%  
> Randy Dunlap| 67| 0.7%  
> Frederic Weisbecker| 66| 0.7%  
> | By changed lines  
> ---  
> Uwe Kleine-Konig| 194249| 18.5%  
> Ralph Campbell| 53250| 5.1%  
> Greg Kroah-Hartman| 31714| 3.0%  
> Stepan Moskovchenko| 30037| 2.9%  
> Arnaud Patard| 28783| 2.7%  
> Mauro Carvalho Chehab| 27902| 2.7%  
> Eliot Blennerhassett| 18490| 1.8%  
> Luis R. Rodriguez| 16555| 1.6%  
> Daniel Mack| 16176| 1.5%  
> Bob Beers| 11703| 1.1%  
> Jason Wessel| 10502| 1.0%  
> Viresh KUMAR| 10499| 1.0%  
> Barry Song| 10116| 1.0%  
> James Chapman| 9645| 0.9%  
> Steve Wise| 9580| 0.9%  
> Sjur Braendeland| 8775| 0.8%  
> Alex Deucher| 7721| 0.7%  
> Jassi Brar| 7554| 0.7%  
> Sujith| 7544| 0.7%  
> Giridhar Malavali| 6867| 0.7%  
  
Uwe Kleine-Konig, who works for Pengutronix, dominates the "changed lines" list due to one patch that Linus just pulled for the 2.5.35-rc5 release that deleted almost all of the ARM default config files. Linus [responded](<https://lwn.net/Articles/395962/>) when Uwe posted his patch with: 

Well, I can hardly refuse a pull that removes almost 200k lines. So I'd happily pull it. Just this single line in your email is a very very powerful thing: 

> 177 files changed, 652 insertions(+), 194157 deletions(-) 

Other than that major cleanup, the majority of the work was in drivers. Ralph Campbell did a lot of Infiniband driver work, I did a lot of cleanup on some staging drivers, and Stepan Moskovchenko and Arnaud Patard contributed new drivers to the staging tree. Mauro Carvalho Chehab contributed lots of Video for Linux driver work -- rounding out the top 6 contributors by lines of code changed. 

Continuing the view that this kernel is much like previous ones, 177 different employers were found to have contributed to the 2.6.35 kernel release: 

> Most active 2.6.35 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1429| 14.2%  
> Red Hat| 1185| 11.8%  
> (Unknown)| 904| 9.0%  
> Intel| 637| 6.3%  
> Novell| 559| 5.6%  
> IBM| 295| 2.9%  
> Nokia| 253| 2.5%  
> (Consultant)| 215| 2.1%  
> Atheros Communications| 175| 1.7%  
> AMD| 173| 1.7%  
> Oracle| 169| 1.7%  
> Samsung| 163| 1.6%  
> Texas Instruments| 162| 1.6%  
> (Academia)| 140| 1.4%  
> Fujitsu| 138| 1.4%  
> Google| 122| 1.2%  
> Renesas Technology| 102| 1.0%  
> Analog Devices| 98| 1.0%  
> Simtec| 96| 1.0%  
> NTT| 93| 0.9%  
> | By lines changed  
> ---  
> Pengutronix| 195175| 18.6%  
> Red Hat| 82334| 7.8%  
> (None)| 79313| 7.6%  
> (Unknown)| 72426| 6.9%  
> QLogic| 72131| 6.9%  
> Novell| 49651| 4.7%  
> Intel| 47260| 4.5%  
> Code Aurora Forum| 40081| 3.8%  
> Mandriva| 29105| 2.8%  
> Atheros Communications| 29055| 2.8%  
> Samsung| 25817| 2.5%  
> ST Ericsson| 20463| 2.0%  
> Analog Devices| 18889| 1.8%  
> AudioScience Inc.| 18545| 1.8%  
> caiaq| 16194| 1.5%  
> Nokia| 14891| 1.4%  
> Texas Instruments| 14864| 1.4%  
> (Consultant)| 14209| 1.4%  
> IBM| 12235| 1.2%  
> ST Microelectronics| 11728| 1.1%  
  
But enough of the normal way of looking at the kernel as a whole body. Let's try something different this time, and break the contributions down by the different functional areas of the kernel. 

The kernel is a bit strange in that it is a mature body of code that still changes quite frequently and throughout the whole body of code. It is not just drivers that are changing, but the "core" kernel as well. That is pretty unusual for a mature code base. The core kernel code -- those files that all architectures and users use no matter what their configuration is -- comprises 5% of the kernel (by lines of code), and you will find that 5% of the total kernel changes happen in that code. Here is the raw number of changes for the "core" kernel files for the 2.6.35-rc5 release. 

> Action| Lines| % of all changes  
> ---|---|---  
> Added | 27,550 | 4.50%  
> Deleted | 7,450 | 1.90%  
> Modified | 6,847 | 4.93%  
  
Note that the percent deleted are a bit off because of the huge defconfig delete by Uwe as described above. 

So, if the changes are made in a uniform way across the kernel, does that mean that the same companies contribute in a uniform way as well, or do some contribute more to one area than another? 

I've broken the kernel files down into six different categories: 

  * **core** : This includes the files in the init, block, ipc, kernel, lib, mm, and virt subdirectories. 
  * **drivers** : This includes the files in the crypto, drivers, sound, security, include/acpi, include/crypto, include/drm, include/media, include/mtd, include/pcmcia, include/rdma, include/rxrpc, include/scsi, include/sound, and include/video subdirectories. 
  * **filesystems** : This includes the files in the fs subdirectory. 
  * **networking** : This includes the files in the net and include/net subdirectories. 
  * **architecture-specific** : This includes the files in the arch, include/xen, include/math-emu, and include/asm-generic subdirectories. 
  * **miscellaneous** : This includes all of the rest of the files not included in the above categories. 

Based on these categories, the size of the 2.6.35 kernel is as follows: 

> Category| % Lines  
> ---|---  
> Core | 4.37%  
> Drivers | 57.06%  
> Filesystems | 7.21%  
> Networking | 5.03%  
> Arch-specific | 21.92%  
> Miscellaneous | 4.43%  
  
Here are the top companies contributing to the different areas of the kernel: 

> Most active 2.6.35 employers (core)  
> ---  
> | By changesets  
> ---  
> Red Hat| 218| 16.5%  
> (None)| 148| 11.2%  
> IBM| 66| 5.0%  
> Novell| 60| 4.5%  
> Intel| 59| 4.5%  
> (Unknown)| 57| 4.3%  
> Fujitsu| 33| 2.5%  
> Google| 30| 2.3%  
> Wind River| 22| 1.7%  
> Oracle| 22| 1.7%  
> Nokia| 22| 1.7%  
> (Consultant)| 22| 1.7%  
> | By lines changed  
> ---  
> Wind River| 9535| 25.4%  
> Red Hat| 6277| 16.7%  
> Novell| 2385| 6.4%  
> (None)| 2074| 5.5%  
> IBM| 2064| 5.5%  
> Intel| 1480| 3.9%  
> Fujitsu| 1431| 3.8%  
> Google| 1417| 3.8%  
> VirtualLogix Inc.| 992| 2.6%  
> ST Ericsson| 957| 2.6%  
> caiaq| 707| 1.9%  
> (Unknown)| 614| 1.6%  
  
The companies contributing to the core kernel files are not surprising. These companies have all contributed to Linux for a long time, and it is part of their core strategy. Wind River has a high number of lines changed due to all of the KGDB work that Jason Wessel has been doing in getting that codebase cleaned up and merged into the main kernel tree. 

> Most active 2.6.35 employers (drivers)  
> ---  
> | By changesets  
> ---  
> (None)| 1022| 18.1%  
> (Unknown)| 678| 12.0%  
> Red Hat| 528| 9.4%  
> Intel| 499| 8.9%  
> Novell| 336| 6.0%  
> Nokia| 199| 3.5%  
> Atheros Communications| 165| 2.9%  
> (Academia)| 94| 1.7%  
> IBM| 86| 1.5%  
> QLogic| 86| 1.5%  
> | By lines changed  
> ---  
> QLogic| 72122| 12.2%  
> (None)| 61356| 10.4%  
> (Unknown)| 60802| 10.3%  
> Red Hat| 47204| 8.0%  
> Intel| 39891| 6.7%  
> Novell| 36951| 6.2%  
> Code Aurora Forum| 34888| 5.9%  
> Mandriva| 28867| 4.9%  
> Atheros Communications| 28844| 4.9%  
> AudioScience Inc.| 18535| 3.1%  
  
Because the drivers make up over 50% of the overall size of the kernel, the contributions here track the overall company statistics very closely. The company AudioScience Inc. sneaks onto the list of changes due to all of the work that Eliot Blennerhassett has been doing on the asihpi sound driver. 

> Most active 2.6.35 employers (filesystems)  
> ---  
> | By changesets  
> ---  
> Red Hat| 134| 15.9%  
> Oracle| 77| 9.1%  
> New Dream Network| 76| 9.0%  
> Novell| 76| 9.0%  
> (Unknown)| 73| 8.7%  
> (None)| 58| 6.9%  
> NetApp| 42| 5.0%  
> Parallels| 39| 4.6%  
> IBM| 23| 2.7%  
> Univ. of Michigan CITI| 23| 2.7%  
> | By lines changed  
> ---  
> Oracle| 7194| 24.2%  
> Red Hat| 6392| 21.5%  
> Novell| 3989| 13.4%  
> (Unknown)| 3081| 10.4%  
> (None)| 2024| 6.8%  
> New Dream Network| 1423| 4.8%  
> NetApp| 897| 3.0%  
> Google| 857| 2.9%  
> Parallels| 687| 2.3%  
> (Consultant)| 546| 1.8%  
  
Filesystem contributions, like drivers, match up with the different company strengths. New Dream Network might not be a familiar name to a lot of people, but their development on the Ceph filesystem pushed it into the list of top contributors. The University of Michigan did a lot of NFS work, bringing that organization into the top ten. 

> Most active 2.6.35 employers (networking)  
> ---  
> | By changesets  
> ---  
> SFR| 74| 9.6%  
> (Consultant)| 73| 9.5%  
> Red Hat| 72| 9.3%  
> (None)| 67| 8.7%  
> ProFUSION| 55| 7.1%  
> Intel| 45| 5.8%  
> Astaro| 35| 4.5%  
> Vyatta| 34| 4.4%  
> (Unknown)| 34| 4.4%  
> Oracle| 20| 2.6%  
> ST Ericsson| 20| 2.6%  
> Univ. of Michigan CITI| 20| 2.6%  
> | By lines changed  
> ---  
> Katalix Systems| 9213| 24.2%  
> ST Ericsson| 8003| 21.0%  
> (Consultant)| 3691| 9.7%  
> Univ. of Michigan CITI| 2334| 6.1%  
> Astaro| 1956| 5.1%  
> Red Hat| 1882| 4.9%  
> Intel| 1607| 4.2%  
> SFR| 1555| 4.1%  
> ProFUSION| 1065| 2.8%  
> (None)| 1060| 2.8%  
> (Unknown)| 1035| 2.7%  
  
Like the filesystem list, networking also shows the University of Michigan's large contributions as well as many of the other common Linux companies. But here a number of not-so-familiar companies start showing up. 

SFR is a French mobile phone company, and contributed lots of changes all through the networking core. ProFUSION is an embedded development company that did a lot of Bluetooth development for this kernel release. Katalix Systems is another embedded development company and they contributed a lot of l2tp changes. Astaro is a networking security company that contributed a number of netfilter changes. 

> Most active 2.6.35 employers (architecture-specific)  
> ---  
> | By changesets  
> ---  
> Red Hat| 146| 7.2%  
> (None)| 143| 7.0%  
> IBM| 120| 5.9%  
> Novell| 109| 5.4%  
> Samsung| 100| 4.9%  
> Texas Instruments| 94| 4.6%  
> AMD| 90| 4.4%  
> Simtec| 85| 4.2%  
> (Unknown)| 75| 3.7%  
> (Consultant)| 73| 3.6%  
> | By lines changed  
> ---  
> Pengutronix| 194211| 60.5%  
> Samsung| 15341| 4.8%  
> ST Microelectronics| 10038| 3.1%  
> (None)| 8338| 2.6%  
> Red Hat| 7981| 2.5%  
> (Consultant)| 6695| 2.1%  
> IBM| 6064| 1.9%  
> Novell| 5973| 1.9%  
> Code Aurora Forum| 5114| 1.6%  
> Analog Devices| 4345| 1.4%  
  
With the architecture-specific files taking up the second largest chunk of code in the kernel, the list of contributing companies is closer to the list of overall contributors as well, with more hardware companies showing that they contribute a lot of development to get Linux working properly on their specific processors. 

> Most active 2.6.35 employers (miscellaneous)  
> ---  
> | By changesets  
> ---  
> Red Hat| 206| 26.9%  
> (None)| 110| 14.4%  
> (Unknown)| 35| 4.6%  
> Novell| 28| 3.7%  
> Intel| 27| 3.5%  
> IBM| 18| 2.4%  
> Fujitsu| 16| 2.1%  
> Google| 15| 2.0%  
> Wind River| 9| 1.2%  
> (Academia)| 9| 1.2%  
> Vyatta| 9| 1.2%  
> | By lines changed  
> ---  
> Red Hat| 12772| 34.0%  
> Broadcom| 6082| 16.2%  
> (None)| 5156| 13.7%  
> (Unknown)| 2757| 7.3%  
> Intel| 2212| 5.9%  
> (Academia)| 1850| 4.9%  
> Samsung| 769| 2.1%  
> Wind River| 593| 1.6%  
> Fujitsu| 592| 1.6%  
> Nokia| 532| 1.4%  
> IBM| 499| 1.3%  
  
The rest of the various kernel files that don't fall into any other major category show that Red Hat has done a lot of work on the userspace performance monitoring tools that are bundled with the Linux kernel. 

As for overall trends in the different categories, Red Hat shows that they completely dominate all areas of developing the Linux kernel when it comes to the number of contributions. No other company shows up in the top ten contributors for all categories like they do. But, by breaking out the kernel contributions in different areas of the kernel, we see that a number of different companies are large contributors in different, important areas. Normally these contributions get drowned out by the larger contributors, but the more specialized contributors are just as important to advancing the Linux kernel.
