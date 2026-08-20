---
title: A group scheduling demonstration
url: https://lwn.net/Articles/433904/
date: "March 16, 2011"
category: "Group scheduling; Scheduler-Group scheduling"
author: "By Jonathan Corbet March 16, 2011"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 16, 2011

There has been much talk of the per-session group scheduling patch which is part of the 2.6.38 kernel, but it can be hard to see that code in action if one isn't doing a 20-process kernel build at the time. Recently, your editor inadvertently got a demonstration of group scheduling thanks to some unexpected results from a Rawhide system upgrade. The way the scheduler works was clearly shown in a way that could be captured at the time. 

Rawhide users know that surprises often lurk behind the harmless-looking `yum upgrade` command. In this particular case, something in the upgrade (related to fonts, possibly) caused every graphical process in the system to decide that it was time to do some heavy processing. The result can be seen in this output from the `top` command: 

> ![\[Top output\]](https://static.lwn.net/images/2011/group-sched-top.png)

The per-session heuristic had put most of the offending processes into a single control group, with the effect that they were mostly competing against each other for CPU time. These processes are, in the capture above, each currently getting 5.3% of the available CPU time. Two processes which were not in that control group were left essentially competing for the second core in the system; they each got 46%. The system had a load average of almost 22, and the desktop was entirely unresponsive. But it was possible to log into the system over the net and investigate the situation without really even noticing the load. 

This isolation is one of the nicest features of group scheduling; even when a large number of processes go totally insane, their ability to ruin life for other tasks on the machine is limited. That, alone, justifies the cost of this feature.
