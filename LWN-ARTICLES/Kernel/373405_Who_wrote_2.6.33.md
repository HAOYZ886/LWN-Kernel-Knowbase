---
title: Who wrote 2.6.33
url: https://lwn.net/Articles/373405/
date: "February 9, 2010"
category: "Releases-2.6.33"
author: "By Jonathan Corbet February 9, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 9, 2010

The release of the [2.6.33-rc7](<http://lwn.net/Articles/373332/>) prepatch indicates that this development cycle is headed toward a close, even if Linus thinks that a -rc8 will be necessary. As has become traditional, LWN has taken a look at some statistics related to this cycle and where the code came from. 

As of this writing, 10,500 non-merge commits have found their way into 2.6.33 - fairly normal by recent standards. These changes added almost 900,000 lines while deleting almost 520,000 others; as a result, the kernel grew by a mere 380,000 lines this time around. According to [the most recent regression list](<http://lwn.net/Articles/373429/>), 97 regressions have been reported in 2.6.33, of which 20 remain unresolved. 

Some 1,152 developers contributed code to 2.6.33. The most active of those were: 

> Most active 2.6.33 developers  
> ---  
> | By changesets  
> ---  
> Ben Hutchings| 145| 1.4%  
> Frederic Weisbecker| 145| 1.4%  
> Arnaldo Carvalho de Melo| 138| 1.3%  
> Luis R. Rodriguez| 130| 1.2%  
> Masami Hiramatsu| 128| 1.2%  
> Bartlomiej Zolnierkiewicz| 124| 1.2%  
> Eric Dumazet| 108| 1.0%  
> Alan Cox| 105| 1.0%  
> Manu Abraham| 102| 1.0%  
> Thomas Gleixner| 101| 1.0%  
> Eric W. Biederman| 97| 0.9%  
> Roel Kluin| 91| 0.9%  
> Alexander Duyck| 88| 0.8%  
> Paul Mundt| 87| 0.8%  
> Johannes Berg| 80| 0.8%  
> Wey-Yi Guy| 77| 0.7%  
> Alex Deucher| 76| 0.7%  
> Jean Delvare| 73| 0.7%  
> Al Viro| 72| 0.7%  
> | By changed lines  
> ---  
> Bartlomiej Zolnierkiewicz| 206468| 18.1%  
> Henk de Groot| 50355| 4.4%  
> Jerry Chuang| 49627| 4.3%  
> Ben Skeggs| 37555| 3.3%  
> Philipp Reisner| 23182| 2.0%  
> Eilon Greenstein| 23123| 2.0%  
> Tomi Valkeinen| 22508| 2.0%  
> Mike Frysinger| 13116| 1.1%  
> Ben Hutchings| 12680| 1.1%  
> Jakob Bornecrantz| 11613| 1.0%  
> Wu Zhangjin| 11325| 1.0%  
> Greg Kroah-Hartman| 10468| 0.9%  
> Rajendra Nayak| 9978| 0.9%  
> Manu Abraham| 9625| 0.8%  
> jack wang| 9171| 0.8%  
> Masami Hiramatsu| 8973| 0.8%  
> Alan Cox| 7672| 0.7%  
> David VomLehn| 7331| 0.6%  
> Arnaldo Carvalho de Melo| 7217| 0.6%  
  
While some of the usual names appear at the top of this list, there are some newcomers as well. Ben Hutchings did a lot of work with network drivers, including the addition of the SolarFlare SFC9000 driver (which has several co-authors). Frederic Weisbecker has been active in a number of areas, adding the hardware breakpoints code, removing the big kernel lock from the reiserfs filesystem, and working with tracing and the perf tool. Arnaldo Carvalho de Melo's work is almost all with the perf events subsystem and the perf tool in particular. Luis Rodriguez continues to work all over the wireless driver subsystem, and with the Atheros drivers in particular, and Masami Hiramatsu's largest contribution is the dynamic probing work. 

In the "lines changed" column, Bartlomiej Zolnierkiewicz continues to work in fixing up some wireless drivers in the staging tree, deleting a lot of code in the process; he also continues his IDE driver work. Henk de Groot added the Agere driver for HERMES II chipsets, Jerry Chuang added the Realtek rtl8192u driver, and Ben Skeggs added much of the Nouveau driver. 

Contributions to 2.6.33 came from 182 employers that your editor was able to identify. The most active of those are: 

> Most active 2.6.33 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1535| 14.6%  
> Red Hat| 1223| 11.6%  
> Intel| 1011| 9.6%  
> (Unknown)| 868| 8.3%  
> IBM| 500| 4.8%  
> Novell| 390| 3.7%  
> Nokia| 319| 3.0%  
> (Consultant)| 316| 3.0%  
> Fujitsu| 204| 1.9%  
> Texas Instruments| 199| 1.9%  
> Atheros Communications| 169| 1.6%  
> (Academia)| 166| 1.6%  
> AMD| 165| 1.6%  
> Oracle| 136| 1.3%  
> Analog Devices| 130| 1.2%  
> Renesas Technology| 126| 1.2%  
> Pengutronix| 125| 1.2%  
> HP| 124| 1.2%  
> Solarflare Communications| 123| 1.2%  
> | By lines changed  
> ---  
> (None)| 304895| 26.7%  
> (Unknown)| 109716| 9.6%  
> Red Hat| 92991| 8.1%  
> Broadcom| 54272| 4.8%  
> Realtek| 49951| 4.4%  
> Intel| 46302| 4.1%  
> Nokia| 37505| 3.3%  
> Novell| 27235| 2.4%  
> IBM| 26783| 2.3%  
> (Consultant)| 25845| 2.3%  
> Texas Instruments| 24232| 2.1%  
> LINBIT| 23247| 2.0%  
> Analog Devices| 19677| 1.7%  
> VMWare| 16045| 1.4%  
> Samsung| 15707| 1.4%  
> Solarflare Communications| 15054| 1.3%  
> JiangSu Lemote Corp.| 11439| 1.0%  
> AMD| 9218| 0.8%  
> Universal Scientific Industrial Co.| 9194| 0.8%  
  
As usual, Red Hat maintains its position at the top of the list, but others are gaining; we may yet see a day when Red Hat is just one of several major contributors. Some readers may be surprised to see Broadcom near the top of the list, given that this company's reputation for contribution is not the best. The truth of the matter is that Broadcom has several developers contributing to various drivers in the networking and SCSI subsystems; it's only in the wireless realm that the trouble starts. 

For the fun of it, your editor typed the "changeset percent" numbers for the last ten releases into a spreadsheet and got this plot: 

![\[Contributor
percentages\]](https://static.lwn.net/images/ns/kernel/empl-contr-pct.png)

The percentages are surprisingly stable over the course of almost three years. The most obviously identifiable trends, perhaps, are the steady increases in the contributions from Intel and Nokia. 

All told, the process continues to function smoothly. The occasional complaint about certain companies not fully participating in the process notwithstanding, the picture is one of hundreds of companies cooperating to a high degree to create the Linux kernel despite their fierce competition elsewhere. The significant percentage of code coming from developers working on their own time shows that Linux is not just a corporate phenomenon, though. We have built a development community which is able to incorporate the interests and work of an astonishingly wide variety of people into a single kernel. 

As always, thanks are due to Greg Kroah-Hartman, who has done a great deal of work to reduce the size of the "(Unknown)" entries in the tables above.
