---
title: Some numbers from the 3.11 development cycle
url: https://lwn.net/Articles/563977/
date: "August 21, 2013"
category: "Releases-3.11"
author: "By Jonathan Corbet August 21, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 21, 2013

As of this writing, the [3.11-rc6](<https://lwn.net/Articles/563900/>) prepatch is out and the 3.11 development cycle appears to be slowly drawing toward a close. That can only mean one thing: it must be about time to look at some statistics from this cycle and see where the contributions came from. 3.11 looks like a fairly typical 3.x cycle, but, as always, there's a small surprise or two for those who look. 

#### Developers and companies

Just over 10,700 non-merge changesets have been pulled into the repository (so far) for 3.11; they added over 775,000 lines of code and removed over 328,000 lines for a net growth of 447,000 lines. So this remains a rather slower cycle than 3.10, which was well past 13,000 changesets by the -rc6 release. As might be expected, the number of developers contributing to this release has dropped along with the changeset count, but this kernel still reflects contributions from 1,239 developers. The most active of those developers were: 

> Most active 3.11 developers  
> ---  
> | By changesets  
> ---  
> H Hartley Sweeten| 333| 3.1%  
> Sachin Kamat| 302| 2.8%  
> Alex Deucher| 254| 2.4%  
> Jingoo Han| 190| 1.8%  
> Laurent Pinchart| 147| 1.4%  
> Daniel Vetter| 137| 1.3%  
> Al Viro| 131| 1.2%  
> Hans Verkuil| 123| 1.1%  
> Lee Jones| 112| 1.0%  
> Xenia Ragiadakou| 100| 0.9%  
> Wei Yongjun| 99| 0.9%  
> Jiang Liu| 98| 0.9%  
> Lars-Peter Clausen| 91| 0.8%  
> Linus Walleij| 90| 0.8%  
> Johannes Berg| 86| 0.8%  
> Tejun Heo| 85| 0.8%  
> Oleg Nesterov| 71| 0.7%  
> Fabio Estevam| 70| 0.7%  
> Tomi Valkeinen| 69| 0.6%  
> Dan Carpenter| 66| 0.6%  
> | By changed lines  
> ---  
> Peng Tao| 260439| 26.9%  
> Greg Kroah-Hartman| 91973| 9.5%  
> Alex Deucher| 55904| 5.8%  
> Kalle Valo| 22103| 2.3%  
> Ben Skeggs| 20282| 2.1%  
> Eli Cohen| 15886| 1.6%  
> Solomon Peachy| 15510| 1.6%  
> Aaro Koskinen| 13443| 1.4%  
> H Hartley Sweeten| 11043| 1.1%  
> Laurent Pinchart| 8923| 0.9%  
> Benoit Cousson| 8734| 0.9%  
> Tomi Valkeinen| 8246| 0.9%  
> Yuan-Hsin Chen| 8222| 0.9%  
> Tomasz Figa| 7668| 0.8%  
> Xenia Ragiadakou| 5136| 0.5%  
> Johannes Berg| 5029| 0.5%  
> Maarten Lankhorst| 4924| 0.5%  
> Marc Zyngier| 4817| 0.5%  
> Hans Verkuil| 4707| 0.5%  
> Linus Walleij| 4379| 0.5%  
  
Someday, somehow, somebody will manage to displace H. Hartley Sweeten from the top of the by-changesets list, but that was not fated to be in the 3.11 cycle. As always, he is working on cleaning up the Comedi drivers in the staging tree — a task that has led to the merging of almost 4,000 changesets into the kernel so far. Sachin Kamat contributed a large set of cleanups throughout the driver tree, Alex Deucher is the primary developer for the Radeon graphics driver, Jingoo Han, like Sachin, did a bunch of driver cleanup work, and Laurent Pinchart did a lot of Video4Linux and ARM architecture work. 

On the "lines changed" side, Peng Tao added the Lustre filesystem to the staging tree, while Greg Kroah-Hartman removed the unloved csr driver from that tree. Alex's Radeon work has already been mentioned; Kalle Valo added the ath10k wireless network driver, while Ben Skeggs continued to improve the Nouveau graphics driver. 

Almost exactly 200 employers supported work on the 3.11 kernel; the most active of those were: 

> Most active 3.11 employers  
> ---  
> | By changesets  
> ---  
> (None)| 976| 9.1%  
> Intel| 970| 9.1%  
> Red Hat| 911| 8.5%  
> Linaro| 890| 8.3%  
> Samsung| 485| 4.5%  
> (Unknown)| 483| 4.5%  
> IBM| 418| 3.9%  
> Vision Engraving Systems| 333| 3.1%  
> Texas Instruments| 319| 3.0%  
> SUSE| 310| 2.9%  
> AMD| 281| 2.6%  
> Renesas Electronics| 265| 2.5%  
> Outreach Program for Women| 230| 2.1%  
> Google| 224| 2.1%  
> Freescale| 151| 1.4%  
> Oracle| 137| 1.3%  
> ARM| 135| 1.3%  
> Cisco| 132| 1.2%  
> | By lines changed  
> ---  
> (None)| 307996| 31.9%  
> Linux Foundation| 93929| 9.7%  
> AMD| 57745| 6.0%  
> Red Hat| 52679| 5.5%  
> Intel| 40868| 4.2%  
> Texas Instruments| 28819| 3.0%  
> Qualcomm| 26215| 2.7%  
> Renesas Electronics| 24084| 2.5%  
> Samsung| 23413| 2.4%  
> Linaro| 20649| 2.1%  
> (Unknown)| 17362| 1.8%  
> IBM| 17337| 1.8%  
> AbsoluteValue Systems| 16872| 1.7%  
> ~~Nokia~~| ~~16847~~|  1.7%  
> Mellanox| 16841| 1.7%  
> Vision Engraving Systems| 12268| 1.3%  
> Outreach Program for Women| 11499| 1.2%  
> SUSE| 10279| 1.1%  
  
Once again, the percentage of changes coming from volunteers (listed as "(None)" above) appears to be slowly falling; it is down from over 11% in 3.10. Red Hat has, for the second time, ceded the top non-volunteer position to Intel, but the fact that Linaro is closing on Red Hat from below is arguably far more interesting. The numbers also reflect the large set of contributions that came in from applicants to the [Outreach Program for Women](<https://wiki.gnome.org/OutreachProgramForWomen>), which has clearly succeeded in motivating contributions to the kernel. 

#### Signoffs

Occasionally it is interesting to look at the Signed-off-by tags in patches in the kernel repository. In particular, if one looks at signoffs by developers other than the author of the patch, one gets a sense for who the subsystem maintainers responsible for getting patches into the mainline are. In the 3.11 cycle, the top gatekeepers were: 

> Most non-author signoffs in 3.11  
> ---  
> | By developer  
> ---  
> Greg Kroah-Hartman| 1212| 12.3%  
> David S. Miller| 801| 8.1%  
> Andrew Morton| 611| 6.2%  
> Mauro Carvalho Chehab| 371| 3.8%  
> John W. Linville| 285| 2.9%  
> Mark Brown| 276| 2.8%  
> Daniel Vetter| 264| 2.7%  
> Simon Horman| 252| 2.6%  
> Linus Walleij| 236| 2.4%  
> Benjamin Herrenschmidt| 172| 1.7%  
> Kyungmin Park| 157| 1.6%  
> James Bottomley| 143| 1.4%  
> Ingo Molnar| 132| 1.3%  
> Rafael J. Wysocki| 131| 1.3%  
> Kukjin Kim| 121| 1.2%  
> Dave Airlie| 121| 1.2%  
> Shawn Guo| 121| 1.2%  
> Felipe Balbi| 119| 1.2%  
> Johannes Berg| 117| 1.2%  
> Ralf Baechle| 110| 1.1%  
> | By employer  
> ---  
> Red Hat| 2156| 21.9%  
> Linux Foundation| 1249| 12.7%  
> Intel| 904| 9.2%  
> Google| 788| 8.0%  
> Linaro| 759| 7.7%  
> Samsung| 429| 4.4%  
> (None)| 408| 4.1%  
> IBM| 332| 3.4%  
> Renesas Electronics| 259| 2.6%  
> SUSE| 249| 2.5%  
> Texas Instruments| 237| 2.4%  
> Parallels| 143| 1.5%  
> Wind River| 126| 1.3%  
> (Unknown)| 124| 1.3%  
> Wolfson Microelectronics| 114| 1.2%  
> Broadcom| 97| 1.0%  
> Fusion-IO| 89| 0.9%  
> OLPC| 87| 0.9%  
> (Consultant)| 86| 0.9%  
> Cisco| 80| 0.8%  
  
We first [looked at signoffs for 2.6.22](<https://lwn.net/Articles/237768/>) in 2007\. Looking now, there are many of the same names on the list — but also quite a few changes. As is the case with other aspects of kernel development, the changes in signoffs reflect the growing importance of the mobile and embedded sector. The good news, as reflected in these numbers, is that mobile and embedded developers are finding roles as subsystem maintainers, giving them a stronger say in the direction of kernel development going forward. 

#### Persistence of code

Finally, it has been some time since we [looked at persistence of code over time](<https://lwn.net/Articles/374574/>); in particular, we examined how much code from each development cycle remained in the 2.6.33 kernel. This information is obtained through the laborious process of running "`git blame`" on each file, looking at the commit associated with each line, and mapping that to the release in which that commit was merged. Doing the same thing now yields a plot that looks like this: 

> ![\[bar
chart\]](https://static.lwn.net/images/2013/ltags-3.11.png)

From this we see that the code added for 3.11 makes up a little over 4% of the kernel as a whole; as might be expected, the percentage drops as one looks at older releases. Still, quite a bit of code from the early 2.6.30's remains untouched to this day. Incidentally, about 19% of the code in the kernel has not been changed since the beginning of the git era; there are still [545 files](<https://lwn.net/Articles/563987/>) that have not been changed at all since the 2.6.12 development cycle. 

Another way to look at things would be to see how many lines from each cycle were in the kernel in 2.6.33 (the last time this exercise was done) compared to what's there now. That yields: 

> ![\[another bar chart\]](https://static.lwn.net/images/2013/ltags-33-3.11.png)

Thus, for example, the 2.6.33 kernel had about 400,000 lines from 2.6.26; of those, about 290,000 remain in 3.11. One other thing that stands out is that the early 2.6.30 development cycles saw fewer changesets merged into the mainline than, say, 3.10 did, but they added more code. Much of that code has since been changed or removed, though. Given that much of that code went into the staging tree, this result is not entirely surprising; the whole point of putting code into staging is to set it up for rapid change. 

Actually, "rapid change" describes just about all of the data presented here. The kernel process continues to absorb changes at a surprising and, seemingly, increasing rate without showing any serious signs of strain. There is almost certainly a limit to the scalability of the current process, but we do not appear to have found it yet.
