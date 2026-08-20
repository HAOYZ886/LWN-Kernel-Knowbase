---
title: "LSFMM: Shrinkers"
url: https://lwn.net/Articles/548092/
date: "April 23, 2013"
category: "Memory management-Shrinkers"
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Dave Chinner and Glauber Costa started this LSFMM memory-management track session by noting that the [lockless lookup work](<https://lwn.net/Articles/419811/>) in the virtual filesystem layer was an improvement but not a complete solution to the VFS scalability problem. In a sense, it has simply split up the work across a set of smaller, but still global locks. The biggest problem has to do with the movement of inodes and directory entries (dentries) on the LRU lists. The global locking used there still does not scale. 

The first step to improve things is to attach those LRU lists to the filesystem superblock structure. The locks are still global, but their scope is now restricted to a single filesystem, reducing contention somewhat. But, they said, things need to be broken up further and it's not clear how that should be done. There is no desire to go to per-CPU lists; the overhead of that solution would be too high. 

There wasn't much discussion of that problem; instead, the bulk of the session was devoted to a related scalability effort: Dave's rework of the shrinker mechanism. Shrinkers are callbacks that can be registered with the memory management subsystem; they are called when memory is tight in the hope that they will free up some resources. There are a number of problems with shrinkers, starting with the fact that they are a global operation while most memory problems are localized to a specific zone or NUMA node. When things get tight, the memory management code calls the shrinkers as a way of pounding on the problem with a large hammer in the hope that things get better. The result is that the rest of the system, which may not be suffering memory shortages, gets hammered in the process. 

In addition, there is little agreement over what a call to a shrinker really means or how the called subsystem should respond. So there's little consistency in how the shrinkers behave and, apparently, some fairly scary code to be found in some of them. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Dave has a solution in the form of a reworked shrinker API. It starts with a generic, per-NUMA-node LRU list that can be adapted to whatever type of object a specific shrinker needs to manage. The shrinker interface itself is built into the list. The result is the wholesale replacement of a lot of custom list management and shrinker code. Shrinkers become a lot more consistent in their effects and the chances of implementers getting things wrong are much reduced. 

Glauber added that he could use this interface in his memory control group ("memcg") work. Reclaim within a memcg is currently limited by the difficulty in finding objects that are specific to the group in question. The current shrinker interface also provides no information on what types of objects to shrink. If more dentries are needed, there is little value in shrinking the inode cache. So it would be nice to have a single entry point that can be adapted to the memory controller's needs. He is not concerned with the per-node capabilities built into the API, but, he said, that API works well and hides the per-node stuff so that he doesn't need to deal with it. 

Mel Gorman agreed that the proposed new shrinker subsystem seemed to be a significant improvement. Shrinkers in their current form, he said, are effectively random; it is best to avoid calling them whenever possible. He did express concerns about the possibility that lock contention could be introduced, but Glauber responded that it is not really issue since the locking hierarchy does not actually change much. 

Johannes Weiner asked why the lists were per-node, instead of being tied to memory management zones. Dave responded that a per-zone approach would just increase the number of lists without adding a lot of benefit; most of the time, when there is a need to free memory, it is sufficient to free it within any zone on the target node. 

The session came to a close with a more general discussion of the nature of shrinker callbacks. It was noted that a lot of kernel developers don't really understand how the interface works, leading to "a lot of abuse." But it also seems that the current API does not match what a lot of users need. The shrinker API exists to ask a subsystem to free a specific amount of memory, but what's often wanted, in both kernel and user space, is a simple indication that memory is getting tight. So it might be nice to provide that kind of signal out of the shrinker code, but it's not clear how that would fit into the new model. 

In any case, the next step is clear: patches are to be posted to the relevant mailing lists for further discussion.
