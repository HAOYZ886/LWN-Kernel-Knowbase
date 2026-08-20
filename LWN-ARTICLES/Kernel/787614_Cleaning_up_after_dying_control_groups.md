---
title: Cleaning up after dying control groups
url: https://lwn.net/Articles/787614/
date: "May 7, 2019"
category: "Control groups; Memory management-Control groups"
author: "By Jonathan Corbet May 7, 2019 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 7, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Control groups are a useful mechanism for managing resource usage in the system, but what happens when the control groups themselves become a resource problem? In a plenary session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, Roman Gushchin described problems he has been facing with deleted control groups that take their time before actually going away. Some of these problems have been fixed, but the issue has not been truly resolved. 

Control groups are managed using a virtual filesystem; a particular group can be deleted just by removing the directory that represents it. But the truth of the matter, Gushchin said, is that while removing a control group's directory hides the group from user space, that group continues to exist in the kernel until all references to it go away. While it persists, it continues to consume resources. 

The problem is especially acute for memory control groups, where every page that is charged to the group holds a reference to it. So a given control group cannot be truly deleted until every page that was charged to it is reclaimed, which can take a long time; if some of those pages are still [![\[Roman Gushchin\]](https://static.lwn.net/images/conf/2019/lsfmm/RomanGushchin-sm.jpg)](<https://lwn.net/Articles/787622/>) actively used, they may avoid reclaim indefinitely. In addition, various bugs have also had the effect of keeping deleted groups around. It all adds up to deleted control groups hanging around and haunting the system; he found 1,500 of them after a week of operation. 

The consequences of this problem are not huge, but still "not nice", he said. Each control group consumes about 200KB of memory while it exists, which begins to add up when thousands of them are waiting to die. All of those groups serve to increase the complexity (and the cost) of traversing the control-group hierarchy in the kernel. That memory use can also throw off memory-management accounting. 

Some of the reasons for the persistence of removed control groups are easier to deal with than others. There was, for example, a rounding error in the handling of user pages that caused the final page not to be reclaimed. This bug showed up in both versions of the control-group subsystem; it has since been fixed. Another issue had to do with the accounting of kernel stacks; it was introduced in the switch to [virtually mapped stacks](<https://lwn.net/Articles/692208/>) in 2016. These stacks were charged to the process (and its group) that first allocated them; when a stack was reused for a new process, the charging was not updated. This problem, too, has been fixed. 

A problem that is not yet fixed has to do with kernel memory obtained from the slab allocators. Many cached objects, such as dentry structures, are obtained from the slab allocator and charged to the appropriate control group; they, too, must be cleaned up before that group can be truly deleted. But when there is not a lot of memory pressure, the shrinkers do not run aggressively and those objects can persist for a long time. Gushchin tried a patch to apply some extra pressure, but it caused performance regressions in the XFS filesystem and was subsequently reverted. So now he is working on a different approach: reparenting slab caches on control-group removal. There is [a patch set](<https://lwn.net/ml/linux-kernel/20190423213133.3551969-1-guro%40fb.com/>) in review, so hopefully this problem will be resolved in the near future. 

With those fixes, the problems that he has observed have been addressed, but there are other potential problems out there. Pages obtained with `vmalloc()` and per-CPU pages are one possible trouble area. In general, though, he said that it is easy to create hidden references to control groups that can impede their removal; this is an area where regressions are likely to happen. 

At the end of the session, Michal Hocko said that the part of the problem is simply the size of structure used to represent a memory control group. Perhaps things could be made a little better by splitting that structure in two and only keeping the core parts when the group is removed. But Johannes Weiner replied that memory pressure is the only thing that is trimming back these deleted groups now; if they are made smaller, they will just pile up more deeply. So, while some manifestations of this problem have been dealt with, the issue of dying control groups will, like the groups themselves, be with us for some time yet.
