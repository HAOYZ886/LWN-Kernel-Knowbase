---
title: "KS2012: memcg/mm: Kernel-memory shrinking"
url: https://lwn.net/Articles/516531/
date: "September 17, 2012"
category: "Memory management-Shrinkers"
author: "By Michael Kerrisk September 17, 2012 2012 Kernel Summit"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Michael Kerrisk**  
September 17, 2012

* * *

[2012 Kernel Summit](<https://lwn.net/Articles/KernelSummit2012/>)

A shrinker is an internal callback routine that the kernel can invoke when memory is tight; a shrinker's job is to reduce the amount of memory used by a specific kernel data structure. During the 2012 Kernel Summit [memcg/mm minisummit](<https://lwn.net/Articles/516439/>), Ying Han talked about [her patch](<https://lwn.net/Articles/516431/>) to implement a per-croup slab shrinker. In the current mainline kernel, reclaiming of kernel memory is not targeted. Ying noted that, to some extent, this did not matter before, but with [Glauber Costa's kernel-memory accounting work](<https://lwn.net/Articles/516529/>), it becomes necessary, in order that memory can be easily reclaimed from a specific cgroup without impacting other cgroups. Most of the introduction talked about the specific technical details of how Ying's patch was implemented. 

Ying's series of patches collides somewhat with some (planned) [work](<https://lwn.net/Articles/456071/>) by Dave Chinner to make shrinkers NUMA-node aware. It remains to be resolved whether Dave's approach should be implemented first. Ying would prefer that just the dentry cache be handled first. (The dentry cache is used to speed the translation of a file pathname to the inode of the file. A consequence of this is that a dentry cache entry "pins" the corresponding inode entry in memory, preventing the inode being freed. This in turn can indirectly keep all the page cache data for that inode--and the inode itself--resident.) Freeing an entry in the dentry cache can in turn free a large amount of memory. Therefore, in Ying's opinion, the dentry cache is the most important problem. Glauber pointed out to Ying that freeing the dentry will make only the inode freeable, but will not necessarily trigger the release of the inode memory. To make that happen will require at least an inode shrinker as well. Unfortunately, Dave Chinner's work is not finished yet, and as it has been planned for a long time, there appears to be a risk that it will never happen or will happen too late to be useful. (Dave was not present at the Kernel Summit to take part in the discussions.) 

During the discussion, the question was raised as to whether Glauber's patches should be merged at all before a per-cgroup slab shrinker is added to the kernel. The concern was that, when a cgroup hits the soft limit, there may be circumstances where memory can't be reclaimed without a per-cgroup slab shrinker. Glauber's response was that, even without the shrinker, his patches can already greatly help some use cases. He considered that development of the code in this area should take an evolutionary approach, providing functionality with limitations now, and improving it later for other use cases. Participants in the room seemed to agree that merging a per-cgroup slab shrinker before Glauber's work probably isn't necessary. 

Andrew Morton felt that it was likely that a more fundamental rework of the memory-management code (i.e., a much bigger reworking even than the piece of work planned by Dave Chinner), should be taken, since, historically, changes to dentry cache management that make any modification to dentry list management have been problematic. Others in the room also seemed to lean to the same view as Andrew, but no final decision was made (changes on a grand scale naturally make memory-management developers nervous about possible bugs and performance regressions, and so they are cautious). Thus, the discussion will have to continue via mail. 

[Next: Improving memory cgroups performance for non-users](<https://lwn.net/Articles/516533/>)
