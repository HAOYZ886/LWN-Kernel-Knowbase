---
title: "LSFMM: Improving the out-of-memory killer"
url: https://lwn.net/Articles/548180/
date: "April 23, 2013"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Few topics have been debated longer in memory management circles than the out-of-memory (OOM) killer. This unloved kernel subsystem is invoked when memory pressure reaches desperate levels; its job is to kill one or more processes in the hope of enabling the system to continue functioning. The problem, as always, is in the choice of which processes to target. At the 2013 LSFMM Summit, Glauber Costa and Greg Thelen led an inconclusive session on how that targeting might be improved. 

The current OOM killer targets processes based on their "OOM score," an indication of both the memory demands created by each process and the system's view of how important the process is; see [this article](<https://lwn.net/Articles/391222/>) for details on the current algorithm. Every process has an `oom_score_adj` value that can be used to tweak its score relative to that of other processes; in this way, the system administrator can implement a policy directing the OOM killer's attention toward (or away from) specific processes. This mechanism falls short of what some administrators would like, though, since it has no awareness of memory control groups. It works in a single, flat namespace of all processes. Memory control groups encapsulate a better understanding of the process tree, but they aren't able to implement OOM policies. 

Jörn Engel noted that the OOM killer as it was initially implemented was quite stupid, but at least it was predictably so. It has gotten quite a bit more complicated since then, to the point that it's hard to know how the system will respond to OOM situations. The result of this complexity is that the OOM killer tends to work well for whoever last tweaked the kernel's target selection policies, but not necessarily for anybody else. 

That led to one of the key questions raised in this session: should OOM handling be moved to user space? That would allow for the creation of arbitrarily complex policies under administrator control. The problem with this approach is that it is tremendously hard to be sure that a user-space OOM daemon will actually be able to do its job when the time comes. One can lock its full address space into memory, but it is difficult to do anything in user space under the constraint that no memory may be allocated at all — and that is just the constraint that will apply in an OOM situation. 

There are also interesting policy questions related to user-space OOM killers. For example, how long should the kernel wait for a user-space OOM daemon to free some memory before taking matters into its own hands? John Stultz pointed out that not everybody sees OOM the same way; Android's low-memory killer wants a notification of a low-memory situation long before it reaches a critical stage, since it needs to be able to clean up processes before they start adversely affecting the performance of the system. 

Glauber would like to see a more flexible way to set policies in the kernel instead. If nothing else, he said, killing a single process might not be the right thing to do; it might be better to just kill all processes contained within a memory control group instead. In that way, an OOM kill would more closely simulate a system crash (from the affected group's point of view) instead of leaving that group limping along without some of its members. 

Mel Gorman said that, no matter what policy is chosen, somebody will always want something different. He saw a few options that the kernel could pursue to try to make more people happy: 

  * Create a global OOM notification mechanism that can be used by processes like the Android low-memory killer. 

  * Create some sort of internal OOM hook in the kernel that would be available to loadable modules or, perhaps, SystemTap scripts. Administrators could then load whatever policy suited them best. 

  * Add a framework by which a policy could be described to the kernel, similar to how firewall rules or packet filters are handled in the network stack. 

From there, the discussion wandered, revisiting a number of the above topics without leading to any real conclusions. It seems likely that the OOM killer will continue to be the subject of debate indefinitely.
