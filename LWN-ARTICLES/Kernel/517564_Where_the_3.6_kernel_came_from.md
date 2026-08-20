---
title: Where the 3.6 kernel came from
url: https://lwn.net/Articles/517564/
date: "September 26, 2012"
category: "Releases-3.6"
author: "By Jonathan Corbet September 26, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 26, 2012

As of this writing, the 3.6 development is nearing its close with the [3.6-rc7](<https://lwn.net/Articles/517380/>) prepatch having been released on September 23\. There may or may not be a 3.6-rc8 before the final release, but, either way, the real 3.6 kernel is not far away. It thus seems like an appropriate time for our traditional look at what happened in this cycle and who the active participants were. 

At the release of -rc7, Linus had pulled 10,153 non-merge changesets from 1,216 developers into the mainline. That makes this release cycle just a little slower than its immediate predecessors, but, with over 10,000 changesets committed, the development community has certainly not been idle. This development cycle is already slightly longer than 3.5 (which required 62 days) and may be as much as two weeks longer by the end, if another prepatch release is required. Almost 523,000 lines of code were added and almost 252,000 were removed this time around for a net growth of about 271,000 lines. 

> Most active 3.6 developers  
> ---  
> | By changesets  
> ---  
> H Hartley Sweeten| 460| 4.5%  
> Mark Brown| 175| 1.7%  
> David S. Miller| 154| 1.5%  
> Axel Lin| 152| 1.5%  
> Johannes Berg| 115| 1.1%  
> Al Viro| 113| 1.1%  
> Hans Verkuil| 111| 1.1%  
> Lars-Peter Clausen| 90| 0.9%  
> Sachin Kamat| 84| 0.8%  
> Daniel Vetter| 83| 0.8%  
> Eric Dumazet| 79| 0.8%  
> Rafael J. Wysocki| 77| 0.8%  
> Guenter Roeck| 76| 0.7%  
> Alex Elder| 76| 0.7%  
> Guennadi Liakhovetski| 75| 0.7%  
> Sven Eckelmann| 75| 0.7%  
> Ian Abbott| 74| 0.7%  
> Arik Nemtsov| 74| 0.7%  
> Dan Carpenter| 72| 0.7%  
> Shawn Guo| 70| 0.7%  
> | By changed lines  
> ---  
> Greg Kroah-Hartman| 113897| 18.3%  
> Mark Brown| 18761| 3.0%  
> H Hartley Sweeten| 14362| 2.3%  
> John W. Linville| 14177| 2.3%  
> Chris Metcalf| 11419| 1.8%  
> Hans Verkuil| 9493| 1.5%  
> Alex Williamson| 7335| 1.2%  
> Pavel Shilovsky| 6226| 1.0%  
> Sven Eckelmann| 5694| 0.9%  
> Johannes Berg| 5518| 0.9%  
> Alexander Block| 5465| 0.9%  
> Kevin McKinney| 5211| 0.8%  
> David S. Miller| 4600| 0.7%  
> Christoph Hellwig| 4512| 0.7%  
> Yan, Zheng| 4481| 0.7%  
> Felix Fietkau| 4433| 0.7%  
> Ola Lilja| 4191| 0.7%  
> Johannes Goetzfried| 4129| 0.7%  
> Vaibhav Hiremath| 4087| 0.7%  
> Nicolas Royer| 3989| 0.6%  
  
H. Hartley Sweeten is at the top of the changesets column this month as the result of a seemingly unending series of patches to get the Comedi subsystem ready for graduation from the staging tree. Mark Brown continues work on audio drivers and related code. David Miller naturally has patches all over the networking subsystem; his biggest contribution this time around was the long-desired removal of the IPv4 routing cache. Axel Lin made lots of changes to drivers in the regulator and MTD subsystems, among others, and Johannes Berg continues his wireless subsystem work. 

Greg Kroah-Hartman pulled the CSR wireless driver into the staging tree to get to the top of the "lines changed" column, even though his 69 changesets weren't quite enough to show up in the left column. John Linville removed some old, unused drivers, making him the developer who removed the most code from the kernel this time around. Chris Metcalf added a number of new features to the Tile architecture subtree. 

The list of developers credited for reporting problems is worth a look: 

> Top 3.6 bug reporters  
> ---  
> Fengguang Wu| 44| 7.7%  
> Martin Hundebøll| 21| 3.7%  
> David S. Miller| 19| 3.3%  
> Dan Carpenter| 17| 3.0%  
> Randy Dunlap| 14| 2.4%  
> Bjørn Mork| 11| 1.9%  
> Al Viro| 10| 1.7%  
> Ian Abbott| 9| 1.6%  
> Stephen Rothwell| 9| 1.6%  
> Eric Dumazet| 8| 1.4%  
  
What we are seeing here is clearly the result of Fengguang Wu's [build and boot testing work](<https://lwn.net/Articles/514278/>). As Fengguang finds problems, he reports them and they get fixed before the wider user community has to deal with them. Coming up with 44 bug reports in just over 60 days is a good bit of work. 

Some 208 companies (that we know of) contributed to the 3.6 kernel. The most active of these were: 

> Most active 3.6 employers  
> ---  
> | By changesets  
> ---  
> (None)| 1124| 11.1%  
> Red Hat| 1035| 10.2%  
> Intel| 884| 8.7%  
> (Unknown)| 828| 8.2%  
> Vision Engraving Systems| 460| 4.5%  
> Texas Instruments| 418| 4.1%  
> Linaro| 409| 4.0%  
> IBM| 286| 2.8%  
> SUSE| 282| 2.8%  
> Google| 243| 2.4%  
> Wolfson Microelectronics| 180| 1.8%  
> (Consultant)| 167| 1.6%  
> Freescale| 152| 1.5%  
> Ingics Technology| 152| 1.5%  
> Samsung| 143| 1.4%  
> Qualcomm| 135| 1.3%  
> Cisco| 127| 1.3%  
> Wizery Ltd.| 125| 1.2%  
> NVidia| 124| 1.2%  
> Oracle| 122| 1.2%  
> | By lines changed  
> ---  
> Linux Foundation| 122520| 19.7%  
> (None)| 63608| 10.2%  
> Red Hat| 59662| 9.6%  
> Intel| 37556| 6.0%  
> (Unknown)| 25719| 4.1%  
> Texas Instruments| 25533| 4.1%  
> Wolfson Microelectronics| 23020| 3.7%  
> Vision Engraving Systems| 14876| 2.4%  
> (Consultant)| 12830| 2.1%  
> Linaro| 11677| 1.9%  
> Tilera| 11436| 1.8%  
> Cisco| 11223| 1.8%  
> IBM| 11006| 1.8%  
> Freescale| 9630| 1.6%  
> SUSE| 9035| 1.5%  
> Marvell| 7984| 1.3%  
> Samsung| 7621| 1.2%  
> OMICRON Electronics| 7259| 1.2%  
> Etersoft| 6236| 1.0%  
> Google| 5673| 0.9%  
  
Greg Kroah-Hartman's move to the Linux Foundation has caused a bit of a shift in the numbers; the Foundation has moved up in the rankings at SUSE's expense. Beyond that, we see the continued growth of the embedded industry's participation, the continuing slow decline of hobbyist contributions, and an equally slow decline in contributions from "big iron" companies like Oracle and IBM. 

Taking a quick look at maintainer signoffs — "Signed-off-by" tags applied by somebody other than the author — the picture is this: 

> Non-author Signed-off-by tags  
> ---  
> | By developer  
> ---  
> Greg Kroah-Hartman| 1232| 14.1%  
> David S. Miller| 754| 8.6%  
> John W. Linville| 376| 4.3%  
> Mauro Carvalho Chehab| 323| 3.7%  
> Mark Brown| 291| 3.3%  
> Andrew Morton| 280| 3.2%  
> Ingo Molnar| 173| 2.0%  
> Luciano Coelho| 132| 1.5%  
> Johannes Berg| 128| 1.5%  
> Gustavo Padovan| 124| 1.4%  
> | By company  
> ---  
> Red Hat| 2323| 26.6%  
> Linux Foundation| 1278| 14.6%  
> Intel| 592| 6.8%  
> Google| 428| 4.9%  
> (None)| 411| 4.7%  
> Texas Instruments| 359| 4.1%  
> Wolfson Microelectronics| 292| 3.3%  
> SUSE| 270| 3.1%  
> Samsung| 230| 2.6%  
> IBM| 189| 2.2%  
  
The last time LWN put up a version of this table was for [2.6.34](<https://lwn.net/Articles/385949/>) in May, 2010. At that time, over half the patches heading into the kernel passed through the hands of somebody at Red Hat or SUSE. That situation has changed a bit since then, though the list of developers contains mostly the same names. Once again, we are seeing the mobile and embedded industry on the rise. 

All told, it looks like business as usual. There are a lot of problems to be solved in the kernel space, so vast numbers of developers are working to solve them. There appears to be little danger that Andrew Morton's famous 2005 [prediction](<https://lkml.org/lkml/2005/9/21/189>) that ""we have to finish this thing one day"" will come true anytime in the near future. But, if we can't manage to finish the job, at least we seem to have the energy and resources to keep trying.
