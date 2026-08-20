---
title: Current issues with memory control groups
url: https://lwn.net/Articles/636331/
date: "March 13, 2015"
category: "Memory management-Control groups"
author: "By Jonathan Corbet March 13, 2015 LSFMM 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
March 13, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

The memory controller for control groups has often been a prominent topic at the annual Linux Storage, Filesystem, and Memory Management Summit. At the 2015 event, control groups were mostly notable by their absence, suggesting that the worst of the problems have been solved. That said, there was time for a brief session where some of the remaining issues were discussed. 

Initially, memory control groups ("memcgs") only tracked user-space memory. Over time, the tracking of kernel-space memory has been added, but, until recently, this feature was acknowledged to not be in particularly good shape. Vladimir Davydov spent quite a lot of time fixing it up, and things work better now. One of the biggest problems was the fact that, while the controller could track and limit kernel memory use, it had no way of reclaiming memory. So, when a particular group hit its limit, things simply came to a stop. Vladimir added per-memcg least-recently-used (LRU) lists for heavily used data structures like dentries and inodes, and kernel-space reclaim now works. 

Much of the remaining discussion centered on whether administrators really need the separate `kmem.limit_in_bytes` knob that controls how much kernel-space memory a control group can use, or whether an overall limit for both kernel-space and user-space memory is sufficient. Michal Hocko noted that kernel-space limits are often used to throttle forking processes, a task that might be better handled in other ways. Perhaps it should be possible to apply ordinary Unix-style resource limits to control groups. Peter Zijlstra said that a number of users want that feature; it will need to be provided or people will continue to propose other control-group-based solutions. 

That left the group without an answer to the question of whether a separate knob for kernel-space memory limits is needed. In the end, there were not a lot of strong feelings on the subject. It will come down to collecting the use cases and seeing whether any are strong enough to warrant adding another knob. 

The final topic discussed was where the biggest holes are in the accounting of kernel memory usage. The most prominent one at this point, it would seem, is tracking the memory used for page tables. So that may be where the next round of memcg development effort is targeted.
