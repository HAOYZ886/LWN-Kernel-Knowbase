---
title: Statistics from the 3.4 development cycle
url: https://lwn.net/Articles/496193/
date: "May 8, 2012"
category: "Releases-3.4"
author: "By Jonathan Corbet May 8, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 8, 2012

With the release of the [3.4-rc6 prepatch](<https://lwn.net/Articles/496079/>), Linus let it be known that he thought the final 3.4 release was probably not too far away. That can only mean one thing: it's time to look at the statistics for this development cycle. 3.4 was an active cycle, with an interesting surprise or two. 

As of this writing, Linus has merged just over 10,700 changes for 3.4; those changes were contributed from 1,259 developers. The total growth of the kernel source this time around is 215,000 lines. The developers most active in this cycle were: 

> Most active 3.4 developers  
> ---  
> | By changesets  
> ---  
> Mark Brown| 284| 2.7%  
> Russell King| 211| 2.0%  
> Johannes Berg| 147| 1.4%  
> Al Viro| 136| 1.3%  
> Axel Lin| 133| 1.2%  
> Johan Hedberg| 122| 1.1%  
> Guenter Roeck| 121| 1.1%  
> Masanari Iida| 109| 1.0%  
> Stanislav Kinsbursky| 97| 0.9%  
> Trond Myklebust| 85| 0.8%  
> Jiri Slaby| 82| 0.8%  
> Ben Hutchings| 82| 0.8%  
> Greg Kroah-Hartman| 78| 0.7%  
> Takashi Iwai| 78| 0.7%  
> Dan Carpenter| 78| 0.7%  
> Stephen Warren| 76| 0.7%  
> Stanislaw Gruszka| 76| 0.7%  
> Alex Deucher| 73| 0.7%  
> | By changed lines  
> ---  
> Joe Perches| 56571| 8.1%  
> Dan Magenheimer| 24077| 3.4%  
> Stephen Rothwell| 17354| 2.5%  
> Greg Kroah-Hartman| 15015| 2.1%  
> Mark Brown| 12266| 1.8%  
> Jiri Olsa| 11842| 1.7%  
> Mark A. Allyn| 10976| 1.6%  
> Stephen Warren| 10386| 1.5%  
> Arun Murthy| 9347| 1.3%  
> Ingo Molnar| 8779| 1.3%  
> Alex Deucher| 8770| 1.3%  
> David Howells| 8034| 1.2%  
> Guenter Roeck| 7634| 1.1%  
> Chris Kelly| 7023| 1.0%  
> Johannes Berg| 6657| 1.0%  
> Ben Hutchings| 6650| 1.0%  
> Al Viro| 6628| 0.9%  
> Russell King| 6610| 0.9%  
  
Mark Brown finds himself at the top of the list of changeset contributors for the second cycle in a row; as usual, he has done a great deal of work with sound drivers and related subsystems. Russell King is the chief ARM maintainer; he has also taken an active role in the refactoring and cleanup of the ARM architecture code. Johannes Berg continues to do a lot of work with the mac80211 layer and the iwlwifi driver, Al Viro has been improving the VFS API and fixing issues throughout the kernel, and Axel Lin has done a lot of cleanup work in the ALSA and regulator subsystems and beyond. 

Joe Perches leads the "lines changed" column with coding-style fixes, [`pr_*()` conversions](<https://lwn.net/Articles/487437/>), and related work. Dan Magenheimer added the "ramster" memory sharing mechanism to the staging tree. Linux-next maintainer Stephen Rothwell made it into the "lines changed" column with the removal of a lot of old PowerPC code. Greg Kroah-Hartman works all over the tree, but the bulk of his changed lines were to be found in the staging tree. 

Some 195 companies contributed changes during the 3.4 development cycle. The top contributors this time around were: 

> Most active 3.4 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1156| 10.8%  
> Intel| 1138| 10.6%  
> Red Hat| 960| 9.0%  
> (Unknown)| 688| 6.4%  
> Texas Instruments| 428| 4.0%  
> IBM| 381| 3.6%  
> Novell| 372| 3.5%  
> (Consultant)| 298| 2.8%  
> Wolfson Microelectronics| 286| 2.7%  
> Samsung| 234| 2.2%  
> Google| 222| 2.1%  
> Oracle| 188| 1.8%  
> Freescale| 175| 1.6%  
> Qualcomm| 161| 1.5%  
> Linaro| 143| 1.3%  
> Broadcom| 140| 1.3%  
> NetApp| 133| 1.2%  
> MiTAC| 133| 1.2%  
> AMD| 132| 1.2%  
> | By lines changed  
> ---  
> (None)| 108509| 15.5%  
> Intel| 67464| 9.7%  
> Red Hat| 65966| 9.4%  
> (Unknown)| 50900| 7.3%  
> IBM| 36800| 5.3%  
> Oracle| 26617| 3.8%  
> Texas Instruments| 25687| 3.7%  
> Samsung| 24966| 3.6%  
> NVidia| 20604| 2.9%  
> Linux Foundation| 16917| 2.4%  
> ST Ericsson| 15792| 2.3%  
> Novell| 15185| 2.2%  
> Wolfson Microelectronics| 14039| 2.0%  
> (Consultant)| 13495| 1.9%  
> AMD| 10151| 1.5%  
> Freescale| 10102| 1.4%  
> Linaro| 9360| 1.3%  
> Google| 9070| 1.3%  
> Qualcomm| 8972| 1.3%  
  
A longstanding invariant in the above table has been Red Hat as the top corporate contributor; in 3.4, however, Red Hat has been pushed down one position by Intel. Red Hat's contributions are down somewhat; 960 changesets in 3.4 compared to 1,290 in 3.3. But the more significant change is the burst of activity from Intel. This work is mostly centered around support for Intel's own hardware, as one would expect, but also extends to things like support for the x32 ABI. Meanwhile, Texas Instruments continues the growth in participation seen over the last few years, as do a number of other mobile and embedded companies. Once upon a time, it was said that Linux development was dominated by "big iron" enterprise-oriented companies; those companies have not gone away, but they are clearly not the only driving force behind Linux kernel development at this point. On the other hand, the participation by volunteers is at the lowest level seen in many cycles, continuing a longstanding trend. 

#### A brief focus on ARM

Recent development cycles have seen a lot of work in the ARM subtree, and 3.4 is no exception; 1,100 changesets touched code in `arch/arm` this time around. Those changes were contributed by 178 developers representing 51 companies. Among those companies, the most active were: 

> Most active 3.4 employers (ARM subtree)  
> ---  
> | By changesets  
> ---  
> (Consultant)| 149| 13.5%  
> Texas Instruments| 121| 11.0%  
> (None)| 103| 9.4%  
> Samsung| 91| 8.3%  
> Linaro| 80| 7.3%  
> NVidia| 54| 4.9%  
> ARM| 52| 4.7%  
> (Unknown)| 48| 4.4%  
> Calxeda| 46| 4.2%  
> Freescale| 40| 3.6%  
> Atmel| 37| 3.4%  
> Atomide| 30| 2.7%  
> OpenSource AB| 24| 2.2%  
> Google| 23| 2.1%  
> ST Ericsson| 23| 2.1%  
> | By lines changed  
> ---  
> Samsung| 8162| 16.8%  
> (None)| 5967| 12.3%  
> NVidia| 4929| 10.2%  
> (Consultant)| 4755| 9.8%  
> Linaro| 3550| 7.3%  
> Texas Instruments| 3118| 6.4%  
> ARM| 2659| 5.5%  
> Calxeda| 2408| 5.0%  
> Atmel| 2080| 4.3%  
> (Unknown)| 1862| 3.8%  
> Vista-Silicon S.L.| 1121| 2.3%  
> Freescale| 1117| 2.3%  
> Atomide| 1005| 2.1%  
> Google| 737| 1.5%  
> PHILOSYS Software| 659| 1.4%  
  
ARM is clearly an active area for consultants, who contributed over 13% of the changes this time around. Otherwise, there are few surprises to be seen in this area; the companies working in the mobile area are the biggest contributors to the ARM tree, while those focused on other types of systems have little presence here. 

There is one other way to look at ARM development. Much of the work on ARM is done through the Linaro consortium. Many developers contributing code from a linaro.com address are "on loan" from other companies; the above table, to the extent possible, credits those changes to the "real" employer that paid for the work. If, instead, all changes from a Linaro address are credited to Linaro, the results change: Linaro, with 11.9% of all the changes in `arch/arm`, becomes the top employer, though it still accounts for fewer changes than independent consultants do. Linaro clearly has become an important part of the ARM development community. 

In summary, it has been another busy and productive development cycle in the kernel community. Despite the usual hiccups, things are stabilizing and chances are good that 3.4-rc7 will be the last prepatch, meaning that this cycle will be a relatively short one. There is little rest for kernel developers, though; the 3.5 cycle with its frantic merge window will start shortly thereafter. Stay tuned to LWN, as always, for ongoing coverage of development in this large and energetic community.
