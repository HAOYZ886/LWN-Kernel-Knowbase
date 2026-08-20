---
title: "KS2012: The memcg/mm minisummit"
url: https://lwn.net/Articles/516439/
date: "September 17, 2012"
category: "Memory management-Conference sessions"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

Day two (28 August) of the 2012 Kernel Summit included [a day-long minisummit](<https://sites.google.com/site/kernelsummit2012/workshops/memcg>) entitled "memcg minisummit" chaired by Ying Han and Johannes Weiner. Ying noted that the original minisummit title was something of a misnomer, since it had grown in scope to cover both [memory control groups (memcg)](<https://lwn.net/Articles/443241/>) and memory-management (mm) topics generally. 

The session began with a statement that it was assumed that everyone in the room was familiar with previous discussions on the topics to be discussed. (Some of these previous discussions took place in the April 2012 LSF/MM meeting. Coverage of that event can be found in LWN articles [here](<https://lwn.net/Articles/490114/>) and [here](<https://lwn.net/Articles/490501/>).) Given the context of the summit, this assumption was considered reasonable by everyone, though readers without a memory-management background may find the record of the discussion a little hard to follow at times. 

Except for one very brief topic, coverage of the various sessions is split out into separate articles. The topics covered were as follows: 

  * [Improving kernel-memory accounting for memory cgroups](<https://lwn.net/Articles/516529/>); some users need better accounting of kernel-memory usage inside cgroups (control groups), in order to to prevent poorly behaved cgroups from exhausting system memory. 

  * [Kernel-memory shrinking](<https://lwn.net/Articles/516531/>); a discussion stemming from Ying Han's patches to implement a per-cgroup slab shrinker. 

  * [Improving memory cgroups performance for non-users](<https://lwn.net/Articles/516533/>); how do we resolve the problem that the current memcg implementation has a performance impact even when memory cgroups are not being used? 

  * [Memory-management performance topics](<https://lwn.net/Articles/516534/>); short discussions of various performance and scalability topics. 

  * [Hierarchical reclaim for memory cgroups](<https://lwn.net/Articles/516535/>); what is the best way to reclaim memory from soft-limited trees of memory cgroups when the system is under memory pressure? 

  * [Reclaiming mapped pages](<https://lwn.net/Articles/516536/>); toward improving reclaim of mapped pages to handle a wider variety of workloads. 

  * [Volatile ranges](<https://lwn.net/Articles/516537/>); looking at various ideas on improving the implementation of this proposed kernel feature. 

  * Memory-management patches work: Michal Hocko briefly discussed the origin of the `memcg-devel` tree. This tree has evolved into being a general memory-management development tree that is not rebased like `linux-next`, but instead takes a mainline release from Linus Torvald's tree and applies Andrew Morton's patches against them. This gives memory-management developers a common, relatively stable ground to implement against. The tree already has a few users and they seem to be happy so far. (Since the meeting, [the tree](<http://git.kernel.org/?p=linux/kernel/git/mhocko/mm.git;a=summary>) has been moved to `kernel.org`, and renamed from `memcg-devel` to `mm`.) 

  * [Moving zcache toward the mainline](<https://lwn.net/Articles/516538/>); what are the barriers to getting the compressed cache feature merged? 

  * [Dirty/writeback LRU](<https://lwn.net/Articles/516539/>); a discussion of Fengguang Wu's proposal to split the file LRU list into clean and dirty lists. 

  * [Proportional I/O controller](<https://lwn.net/Articles/516540/>); two proposed solutions to improve its performance for cgroup workloads. 

  * [Shared-memory accounting in memory cgroups](<https://lwn.net/Articles/516541/>); dealing with some scenarios where memory cgroups are unfairly charged for memory usage. 

  * [NUMA scheduling](<https://lwn.net/Articles/516542/>); a discussion of competing patch sets that implement this feature. 

By and large, this was considered a successful meeting by the memory-management developers in attendance. Ying Han kept everyone on track and the meeting to schedule, and each of the topics were discussed in detail; good progress was made on many issues, and the participants gained insights into several issues that will affect an increasing number of users in the future. Hopefully, some of the remaining issues will now be more easily resolved on mailing lists. 

[Michael Kerrisk would like to thank Fengguang Wu, Glauber Costa, Johannes Weiner, Michal Hocko, and especially Mel Gorman for assistance with the write-up of the minisummit.]
