---
title: Statistics for the 3.3 development cycle
url: https://lwn.net/Articles/485058/
date: "March 6, 2012"
category: "Releases-3.3"
author: "By Jonathan Corbet March 6, 2012"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
March 6, 2012

As of this writing, the 3.3 development cycle is at [3.3-rc6](<https://lwn.net/Articles/485101/>) and things are starting to look pretty stable. So it must be about time for our traditional summary of interesting statistics for the 3.3 kernel. It has been an active cycle, with some 10,350 changesets merged from just over 1,200 developers. Some 563,000 lines of code were added to the kernel, but 395,000 lines were removed, for a net growth of about 168,000 lines. 

The most active developers this time around were: 

> Most active 3.3 developers  
> ---  
> | By changesets  
> ---  
> Mark Brown| 281| 2.7%  
> Mauro Carvalho Chehab| 271| 2.6%  
> Axel Lin| 240| 2.3%  
> Al Viro| 200| 2.0%  
> Tejun Heo| 123| 1.2%  
> Tomi Valkeinen| 101| 1.0%  
> Russell King| 100| 1.0%  
> Matthew Wilcox| 99| 1.0%  
> Ben Skeggs| 94| 0.9%  
> Johannes Berg| 93| 0.9%  
> Stanislaw Gruszka| 93| 0.9%  
> Kuninori Morimoto| 92| 0.9%  
> Eliad Peller| 90| 0.9%  
> Takashi Iwai| 88| 0.9%  
> Eric Dumazet| 87| 0.8%  
> Dan Carpenter| 86| 0.8%  
> Franky Lin| 77| 0.8%  
> Kalle Valo| 74| 0.7%  
> Lars-Peter Clausen| 73| 0.7%  
> Artem Bityutskiy| 68| 0.7%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 88664| 11.7%  
> Stanislaw Gruszka| 38012| 5.0%  
> Mathieu Desnoyers| 25968| 3.4%  
> Mauro Carvalho Chehab| 21063| 2.8%  
> Alan Cox| 20948| 2.8%  
> Kumar Gala| 12083| 1.6%  
> Aurelien Jacquiot| 9998| 1.3%  
> Mark Brown| 9208| 1.2%  
> Evgeniy Polyakov| 7979| 1.1%  
> David Daney| 7684| 1.0%  
> Manuel Lauss| 7316| 1.0%  
> Kuninori Morimoto| 7115| 0.9%  
> Dmitry Kasatkin| 6880| 0.9%  
> Jussi Kivilinna| 6861| 0.9%  
> Ben Skeggs| 6699| 0.9%  
> Axel Lin| 6251| 0.8%  
> Jesse Gross| 5940| 0.8%  
> Takashi Iwai| 5140| 0.7%  
> Rob Clark| 4962| 0.7%  
> Bart Van Assche| 4711| 0.6%  
  
Mark Brown regularly appears in the list of top contributors; for 3.3, he contributed large numbers of patches in the sound and multi-function device subsystems. Mauro Carvalho Chehab is usually better known for routing vast numbers of Video4Linux2 changes into the kernel; this time, he wrote a substantial portion of those patches himself. Axel Lin's contributions were also in the sound subsystem. So, in other words, the top three contributors to the 3.3 kernel were all working with multimedia which is, clearly, a area with a lot of development going on. Al Viro is _not_ a media developer; his work was mostly cleaning up interfaces deep within the virtual filesystem layer. Tejun Heo continues to dig into code all over the kernel; this time around he fixed up the memblock allocator, made a number of process freezer improvements, reworked the CFQ block I/O scheduler, and made a number of control group changes. 

In the lines-changed column, Greg Kroah-Hartman heads the list again. This time around, almost all of his changes were deletions; a lot of code was removed from staging this time around, often because it graduated to the mainline kernel. Stanislaw Gruszka made a lot of changes to the iwlegacy network driver. Mathieu Desnoyers made the list for having added the LTTng tracing subsystem; unfortunately, that code was subsequently [removed](<https://lwn.net/Articles/471973/>) and will not appear in the 3.3 release. Alan Cox made the top five for his work with the gma500 graphics driver and its move out of the staging tree. 

Just over 200 companies have been identified as having supported contributions to the 3.3 kernel. The most active companies this time around were: 

> Most active 3.3 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1322| 12.9%  
> Red Hat| 1290| 12.6%  
> Intel| 897| 8.8%  
> (Unknown)| 524| 5.1%  
> Novell| 450| 4.4%  
> Texas Instruments| 422| 4.1%  
> IBM| 357| 3.5%  
> Wolfson Microelectronics| 282| 2.8%  
> Qualcomm| 249| 2.4%  
> (Consultant)| 243| 2.4%  
> MiTAC| 240| 2.3%  
> Broadcom| 231| 2.3%  
> Samsung| 216| 2.1%  
> Google| 211| 2.1%  
> Oracle| 183| 1.8%  
> Freescale| 163| 1.6%  
> Wizery Ltd.| 111| 1.1%  
> Parallels| 108| 1.1%  
> Renesas Electronics| 104| 1.0%  
> (Academia)| 102| 1.0%  
> | By lines changed  
> ---  
> Novell| 113910| 15.1%  
> Red Hat| 111338| 14.7%  
> (None)| 81133| 10.7%  
> Intel| 68378| 9.0%  
> Texas Instruments| 35696| 4.7%  
> Samsung| 27220| 3.6%  
> EfficiOS| 25990| 3.4%  
> Freescale| 22266| 2.9%  
> (Unknown)| 19307| 2.6%  
> (Consultant)| 18529| 2.5%  
> IBM| 16026| 2.1%  
> Wolfson Microelectronics| 13688| 1.8%  
> Qualcomm| 11736| 1.6%  
> Broadcom| 11180| 1.5%  
> Mellanox| 8856| 1.2%  
> Cavium| 7903| 1.0%  
> Renesas Electronics| 7574| 1.0%  
> Google| 7135| 0.9%  
> MiTAC| 6491| 0.9%  
> Nicira Networks| 6004| 0.8%  
  
This table has yielded few surprises in recent years; for the most part, the companies listed here remain the same from one cycle to the next. The continued growth in contributions from companies in the mobile and embedded areas is worth calling out, though. These companies are not just contributing support for their hardware; increasingly, they are also contributing to the core kernel and driving its evolution in the directions needed for their particular market. Once upon a time, it was common to hear that Linux kernel development was dominated by the needs of large enterprise deployments; few people make that claim now. 

One other trend your editor has noted over time is a slow decline in the percentage of changes coming from people working on their own time. Here is a chart showing the numbers for all kernels since 2.6.25: 

![\[Volunteer contributions\]](https://static.lwn.net/images/2012/volunteer-contributions.png)

The numbers are somewhat noisy, but the trend over the last four years suggests that volunteers are not contributing as much as they once were. It is unclear why that might be. One possibility is that the kernel has reached a point where there are few easy jobs left; the complexity of contemporary kernel development may be discouraging volunteers. Or it may simply be that anybody who demonstrates an ability to get code into the kernel tends not to remain a volunteer for long unless that is what they really want to be; all the rest end up getting hired. The truth may be a combination of both - or something else altogether. 

Volunteer developers are important; they help tie the kernel to the wider community and some of them will become next year's professional developers and subsystem maintainers. A kernel that is unattractive to volunteers may find itself short of developers in the future. Thus far, there is nothing to suggest that any such developer shortage is happening; the 3.3 kernel, with 1,200 contributors, is as strong as any in that regard. That said, this trend is worth watching. 

As a whole, though, the kernel remains a fast-paced and seemingly healthy project. The 3.3 release should happen sometime in mid-March, right on schedule. There is already a lot of interesting code lining up for merging in 3.4; expect to see another set of big numbers when the 3.4 version of this article appears in roughly 80 days time.
