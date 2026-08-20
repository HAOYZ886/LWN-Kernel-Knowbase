---
title: Who wrote 3.5
url: https://lwn.net/Articles/507986/
date: "July 25, 2012"
category: "Releases-3.5"
author: "July 25, 2012 This article was contributed by Greg Kroah-Hartman ."
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

July 25, 2012

This article was contributed by [Greg Kroah-Hartman](<http://www.kroah.com/linux/>).

Now that the 3.5 Linux kernel has been [released](<https://lwn.net/Articles/507729/>), it's time for the traditional look at who wrote it. Here we'll try to summarize who did all of the work that went into this release. 

#### Fastest-changing kernel ever

The 3.5 kernel was released one day faster than the 3.4 kernel was, in 62 days. The last time a kernel was released this quickly was back in 2005 with the 2.6.14 kernel release (61 days). 

In those 62 days, the kernel developers crammed in a record-breaking 176.73 changes per day (7.36 changes per hour.) This is the fastest-changing kernel that has been recorded since I started keeping track of this development metric back in the 2.5 kernel release series. 

These changes resulted in the following overall changes: 

> Changes in 3.5  
> ---  
> 571987 | lines added  
> 358836 | lines removed  
> 135848 | lines modified  
  
The kernel is still increasing at a pretty constant 1.37% growth in the number of lines and files, which is similar to the growth rate of the past three kernel releases. 

#### Individual contributions

1,195 different developers contributing patches to the 3.5 kernel; those developers worked for at least 194 different companies. The names of the contributing developers are pretty familiar to those who track these statistics: 

> Most active 3.5 developers  
> ---  
> | By changesets  
> ---  
> Greg Kroah-Hartman| 239| 2.2%  
> Axel Lin| 191| 1.7%  
> Mark Brown| 187| 1.7%  
> H. Hartley Sweeten| 135| 1.2%  
> David S. Miller| 131| 1.2%  
> Daniel Vetter| 130| 1.2%  
> Al Viro| 128| 1.2%  
> Stephen Warren| 121| 1.1%  
> Tejun Heo| 112| 1.0%  
> Eric Dumazet| 105| 1.0%  
> Hans Verkuil| 102| 0.9%  
> Paul Mundt| 102| 0.9%  
> Johannes Berg| 102| 0.9%  
> Shawn Guo| 102| 0.9%  
> Thomas Gleixner| 98| 0.9%  
> Dan Carpenter| 86| 0.8%  
> Sam Ravnborg| 84| 0.8%  
> Chris Wilson| 79| 0.7%  
> Trond Myklebust| 74| 0.7%  
> Eric W. Biederman| 73| 0.7%  
> Jiri Slaby| 73| 0.7%  
> Arnaldo Carvalho de Melo| 71| 0.6%  
> Artem Bityutskiy| 68| 0.6%  
> Hans de Goede| 68| 0.6%  
> Takashi Iwai| 64| 0.6%  
> | By changed lines  
> ---  
> Paul Gortmaker| 44000| 5.7%  
> Viresh Kumar| 20425| 2.7%  
> Steven Rostedt| 14615| 1.9%  
> H. Hartley Sweeten| 13083| 1.7%  
> Dave Airlie| 12217| 1.6%  
> Sakari Ailus| 10835| 1.4%  
> Dong Aisheng| 10574| 1.4%  
> Sonic Zhang| 10494| 1.4%  
> Paul Walmsley| 10084| 1.3%  
> Ben Skeggs| 10000| 1.3%  
> Rob Herring| 9886| 1.3%  
> Sascha Hauer| 9602| 1.3%  
> Stephen Warren| 9365| 1.2%  
> Parav Pandit| 8846| 1.2%  
> Nicholas Bellinger| 8704| 1.1%  
> Linus Walleij| 8496| 1.1%  
> Shawn Guo| 7797| 1.0%  
> David S. Miller| 7445| 1.0%  
> Phil Edworthy| 7189| 0.9%  
> Sam Ravnborg| 6752| 0.9%  
> Hans Verkuil| 6718| 0.9%  
> Alexander Shishkin| 6668| 0.9%  
> Tejun Heo| 6579| 0.9%  
> Greg Kroah-Hartman| 6524| 0.9%  
> Vladimir Serbinenko| 6451| 0.8%  
  
In the quantity category (remember, we don't judge quality), I did a large number of cleanup patches removing old USB logging macros from the system, which resulted in the majority of my changes in the 3.5 kernel. Axel contributed a great number of regulator driver fixes and enhancements, and Mark Brown did the majority of his work in the sound system-on-a-chip drivers area. H. Hartley Sweeten has been working on cleaning up the Comedi (data acquisition) drivers to get them ready to move out of the staging area of the kernel. This work has him showing up in these statistics for the first time. And rounding out the top five is David Miller with a large number of networking core and driver patches. 

Along with H. Hartley Sweeten, Daniel Vetter is also a newcomer to the "top changesets" list. His contributions came from numerous changes and enhancements to the Intel graphics drivers. Although Hans Verkuil is also a name that might not be familiar to many, his contributions to the Video4Linux drivers and core code show he is a core contributor to a subsystem that many users rely on every day. 

Considering the statistics in lines changed, Paul Gortmaker leads by virtue of the fact that he deleted all of the old Token Ring drivers from the kernel. Viresh Kumar did a lot of SPEAr processor and driver work, adding numerous new drivers for the platform. Steven Rostedt did a large amount of development on ftrace and ktest (a kernel-testing tool). H. Hartley Sweeten did the aforementioned Comedi driver cleanup work, and Dave Arlie made major changes in the area of graphics drivers. 

#### Reviewing the work

All kernel patches are reviewed and "Signed-off-by" a subsystem maintainer before they are committed to the Linux kernel. The developers with the most sign-offs for the 3.5 kernel were as follows: 

> Developers with the most signoffs (total 20391)  
> ---  
> Greg Kroah-Hartman| 1216| 6.0%  
> David S. Miller| 922| 4.5%  
> Mauro Carvalho Chehab| 605| 3.0%  
> Mark Brown| 549| 2.7%  
> John W. Linville| 493| 2.4%  
> Linus Torvalds| 424| 2.1%  
> Andrew Morton| 373| 1.8%  
> Daniel Vetter| 268| 1.3%  
> Dave Airlie| 255| 1.3%  
> Al Viro| 197| 1.0%  
> Axel Lin| 191| 0.9%  
> Trond Myklebust| 173| 0.8%  
> Arnaldo Carvalho de Melo| 165| 0.8%  
> James Bottomley| 164| 0.8%  
> Artem Bityutskiy| 157| 0.8%  
> Kyungmin Park| 156| 0.8%  
> Samuel Ortiz| 154| 0.8%  
> Linus Walleij| 153| 0.8%  
> Ingo Molnar| 150| 0.7%  
> Wey-Yi W Guy| 146| 0.7%  
> Thomas Gleixner| 139| 0.7%  
> Stephen Warren| 136| 0.7%  
> H. Hartley Sweeten| 135| 0.7%  
> Shawn Guo| 131| 0.6%  
> Paul Mundt| 128| 0.6%  
  
I ended up doing the most sign-offs for this kernel release because of many changes in the staging and USB subsystems. David Miller follows with his work in the networking and networking driver trees, as well as in the IDE drivers. Mauro is the maintainer of the Video4Linux subsystem, Mark Brown is the maintainer of the embedded sound drivers, and John Linville is the maintainer of the wireless driver subsystem. 

These numbers reflect the picture of what has been happening in the past few kernel releases, with the majority of changes happening in the staging and networking areas of the kernel. 

#### Who sponsored this work

Here is the list of the companies who sponsored the developers doing the work for this kernel release, and the number of changes attributed to them: 

> Top changeset contributors by employer  
> ---  
> (None)| 1343| 12.3%  
> Red Hat| 1123| 10.2%  
> Intel| 1061| 9.7%  
> (Unknown)| 860| 7.8%  
> Linaro| 519| 4.7%  
> Novell| 440| 4.0%  
> Texas Instruments| 313| 2.9%  
> IBM| 282| 2.6%  
> Linux Foundation| 279| 2.5%  
> Google| 265| 2.4%  
> Samsung| 251| 2.3%  
> Oracle| 204| 1.9%  
> Renesas Electronics| 201| 1.8%  
> MiTAC| 191| 1.7%  
> NVIDIA| 188| 1.7%  
> Wolfson Microelectronics| 187| 1.7%  
> (Consultant)| 160| 1.5%  
> NetApp| 153| 1.4%  
> Vision Engraving Systems| 135| 1.2%  
> Qualcomm| 121| 1.1%  
  
Longtime readers of this series of articles will notice that Linaro has appeared in the top 5 kernel developer companies by number of contributions for the first time. This is due to the increased number of patches Linaro has been contributing, as well as the organization's wish to have the member company employees' contributions be counted as coming from Linaro, instead of the member company itself, as we had previously been doing. 

A newcomer to the top 20 companies is Vision Engraving Systems, thanks to the Comedi development work from H. Hartley Sweeten. With his work, hopefully this subsystem can move out of the staging area of the kernel in a future release. 

Other than the large jump from Linaro, the other companies in the top 25 are well known. Even NVIDIA--despite Linus's well-publicized, and in my opinion well-deserved, [criticism](<http://arstechnica.com/information-technology/2012/06/linus-torvalds-says-f-k-you-to-nvidia/>) of its Linux graphics driver development efforts--continues to be a large contributor to the kernel in the area of embedded processor support for its products. Texas Instruments, Samsung, MiTAC, Wolfson Microelectronics, Qualcomm, Renesas, and Nokia are also primarily focused in the embedded Linux area, showing the wide range of ongoing company support for Linux in embedded systems. 

#### Work continues as usual

With the 3.5 kernel release, the number of contributors remains as high as previous releases, the rate of contributions is greater than ever (as measured by number of patches per day), and the rate of increase in the size of the kernel code remains the same as it has been for the past year. This shows that the kernel development community is still growing, and maintaining its incredibly rapid development cycle, ensuring that Linux remains the largest software engineering project ever.
