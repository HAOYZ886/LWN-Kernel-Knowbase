---
title: Merging zswap
url: https://lwn.net/Articles/551401/
date: "May 22, 2013"
category: Transcendent memory; zswap
author: "By Jonathan Corbet May 22, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 22, 2013

As reported in our [Linux Storage, Filesystem, and Memory Management Summit coverage](<https://lwn.net/Articles/LSFMM2013/>), the decision was made to merge the [zswap](<https://lwn.net/Articles/537422/>) compressed swap cache subsystem while holding off on the rather more complex "zcache" subsystem. But conference decisions can often run into difficulties during the implementation process; that has proved to be the case here. 

Zswap developer Seth Jennings duly [submitted the code](<https://lwn.net/Articles/549740/>) for consideration for the 3.11 development cycle. He quickly ran into opposition from zcache developer Dan Magenheimer; Dan had agreed with the merging of zswap in principle, but he [expressed concerns](<https://lwn.net/Articles/551423/>) that zswap may perform poorly in some situations. According to Dan, it would be better to fix these problems before merging the code: 

I think the real challenge of zswap (or zcache) and the value to distros and end users requires us to get this right BEFORE users start filing bugs about performance weirdness. After which most users and distros will simply default to 0% (i.e. turn zswap off) because zswap unpredictably sometimes sucks. 

The discussion went around in circles the way that in-kernel compression discussions often do. In the end, though, the consensus among memory management developers (but not Dan) was probably best [summarized](<https://lwn.net/Articles/551424/>) by Mel Gorman: 

I think there is a lot of ugly in there and potential for weird performance bugs. I ran out of beans complaining about different parts during the review but fixing it out of tree or in staging like it's been happening to date has clearly not worked out at all. 

So the end result is likely to be that zswap will be merged for 3.11, but with a number of warnings attached to it. Then, with luck, the increased visibility of the code will motivate developers to prepare patches and improve the code to a point where it is production-ready.
