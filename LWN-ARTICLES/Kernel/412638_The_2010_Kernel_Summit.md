---
title: The 2010 Kernel Summit
url: https://lwn.net/Articles/412638/
date: "November 2, 2010"
category: Kernel Summit
author: "By Jonathan Corbet November 2, 2010 2010 Kernel Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 2, 2010

* * *

[2010 Kernel Summit](<https://lwn.net/Articles/KernelSummit2010/>)

The 2010 Kernel Summit was held on November 1 and 2 in Cambridge, MA, USA. Some seventy or so top-level kernel developers gathered there to discuss a wide range of topics which are of interest to the wider kernel community. Your editor was there, frantically taking notes. Reports from the first day's sessions can be found below: 

  * [Welcoming newcomers](<https://lwn.net/Articles/412639/>): is the kernel development community sufficiently open to newcomers to ensure an adequate flow of new developers? If not, what can we do about it? 

  * [ABI status for tracepoints](<https://lwn.net/Articles/412685/>). There is an increasing amount of instrumentation which depends on tracepoints; they are becoming part of the kernel binary interface. To what extent should tracepoints have set-in-cement ABI status? 

  * [The core kernel vision](<https://lwn.net/Articles/412687/>). Neil Brown asks: do we have a core vision for how the kernel should be developed? If so, how do we enforce it? 

  * [A staging process for ABIs](<https://lwn.net/Articles/412744/>). Getting user-space ABIs right is hard; should there be a process for tentatively adding interfaces which are subject to change? 

  * [Deadline scheduling](<https://lwn.net/Articles/412745/>): does the kernel need a new class for deadline scheduling? 

  * [Regressions](<https://lwn.net/Articles/412746/>) as seen by kernel regression tracker Rafael Wysocki. 

  * [Performance regressions](<https://lwn.net/Articles/412747/>): performance-sensitive users often notice that kernel releases tend to get slower over time. What can we do about that? 

  * [Big out-of-tree projects](<https://lwn.net/Articles/412748/>): are they a problem, and what can be done about them? 

  * [Checkpoint/restart](<https://lwn.net/Articles/412749/>): what are its prospects for inclusion? 

  * [Lightning talks](<https://lwn.net/Articles/412750/>): the final session of the day was dedicated to short talks on Coccinelle, the device model, the big kernel lock, and more. 

The sessions which were held on the second day of the summit are: 

  * [Linux at NASDAQ](<https://lwn.net/Articles/412818/>); a session on how a high-volume end users uses Linux and where the pain points are. 

  * [Scalability](<https://lwn.net/Articles/412847/>): where we stand and what comes next. 

  * [Minisummit reports](<https://lwn.net/Articles/413036/>) covering networking, filesystems, Video4Linux, embedded, power management, and more. [![\[Group photo\]](https://static.lwn.net/images/2010/ks2010-sm.jpg)](<https://lwn.net/Articles/412889/>)

  * [Security](<https://lwn.net/Articles/413102/>): are we doing enough to keep the kernel secure? 

  * [Scheduling issues](<https://lwn.net/Articles/413046/>): this session was essentially a second end-user presentation focused on Google's scheduling challenges. 

  * [Kernel.org update](<https://lwn.net/Articles/413059/>): the current status of the infrastructure behind kernel development. 

  * A stable tree update from Greg Kroah-Hartman. The bulk of the information presented here was also seen at Greg's [LinuxCon Japan keynote](<https://lwn.net/Articles/407525/>), so readers may want to go there for the details. Beyond that, Greg noted that he will start dropping trees a little sooner (2.6.35 is about to get its last update). There were some questions on the routing of stuff to stable - both in terms of missing important patches and sending stuff which shouldn't go there. The solution in both cases is for maintainers to pay more attention. 

  * [Development process issues](<https://lwn.net/Articles/413061/>): Linus Torvalds and Andrew Morton talk about how the process is going, what can be improved, and whether the version numbering scheme should change. 

  * [Future summits](<https://lwn.net/Articles/413095/>): the format of the kernel summit looks likely to change starting in 2011. 

The Kernel Summit was followed by a joint reception with the Linux Plumbers Conference. An election for the Linux Foundation's Technical Advisory Board was held there. The five open seats were won by James Bottomley and Chris Mason (both incumbents), joined by newcomers John Linville, Grant Likely, and Hugh Blemings.
