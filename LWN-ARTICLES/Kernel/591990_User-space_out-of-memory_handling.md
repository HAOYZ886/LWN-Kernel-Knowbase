---
title: "User-space out-of-memory handling"
url: https://lwn.net/Articles/591990/
date: "March 26, 2014"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

While opinions on how the kernel should respond to out-of-memory (OOM) situations vary, almost everybody seems to agree that what the kernel does now is in need of improvement. A session on the topic during the memory management track at the 2014 Linux Storage, Filesystem, and Memory Management Summit covered some possible improvements, but reached no real conclusions. 

David Rientjes used the session to talk about his user-space OOM handling patches and to ask for a green light for their inclusion. He spent a while talking about how these patches work; this introduction can be found in [David's article on the subject](<https://lwn.net/Articles/590960/>) and will not be [![\[David Rientjes\]](https://static.lwn.net/images/conf/2014/lsfmm/DavidRientjes-sm.jpg)](<https://lwn.net/Articles/592095/>) repeated here. David has been pushing this work for the last year or so, but it seems clear that the community is still not completely sold on it. 

Sasha Levin asked whether it might be better to use the [vmpressure](<https://lwn.net/Articles/524742/>) mechanism, which sends notifications when memory is getting tight, rather than waiting for a full OOM situation and hoping that user space can handle it. The problem with that approach, as Rik van Riel put it, is that there is no limit to how quickly a system can consume its memory. David added that the vmpressure mechanism does not work as well as one might think. As an illustration of the problem, consider a process that locks many pages into memory; it will consume much of the available memory, but no pressure notifications will result because no reclaim is yet happening. The system can then go from a "no pressure" state to "out of memory" almost instantaneously once reclaim starts; there simply is no opportunity for user space to respond. 

As the discussion went on, it became clear that the most discomfort existed around the use of a user-space handler to deal with global OOM situations. If a single control group under the memory controller (a "memcg") runs out of memory, it makes sense to have a user-space handler respond. But, Michal Hocko asked, do we really want to handle global OOM situations (where the system as a whole is out of memory) in user space? He agreed that the current code does not work for everybody, but, he said, pushing responsibility into user space opens up a can of worms and would be hard to maintain in the long term. It would be better, he suggested, to improve the global OOM killer in the kernel instead. 

Tim Hockin, speaking about his work at Google (which has driven the user-space OOM handler development), talked about the problems they have had with OOM-handling requirements that have changed over time. Google has a hard time deciding what it wants to have happen in OOM situations; it seems hard to expect the kernel developers to anticipate where those requirements might go in the future. That has led to the desire to push the policy into user space where it can be changed without the need to build and deploy a new kernel — a process which does not happen quickly at Google. He would be happy with an in-kernel mechanism that allowed policies to be changed, but only if it is possible to effect a change without building a new kernel. 

Robert Haas agreed that moving the policy into user space gives users the ability to make changes without having to change the kernel itself. Kernel developers, he said, simply are not smart enough to come up with all possible policies. But David said he was willing to try if that was how it would be done, though he suggested that the community might not be happy about the "hundreds of patches" implementing all of the possible policies that would result. 

There was also some unhappiness about David's use of the memcg mechanism for global OOM handling. That mechanism will only work if control groups are built into the kernel, but there are still plenty of users who prefer not to enable control groups at all. The motivation for using that interface was to allow per-memcg and global OOM handlers to work with the same interface and be coded the same way. Peter Zijlstra suggested that the same control files could be placed in `/proc` for global OOM handling, providing something very close to the same interface without needing to enable control groups. 

David asked for some guidance on how he could make progress in this area. It has been hard to get a consensus on his user-space OOM handling patches, but no viable alternatives have come forward. So he is somewhat stuck. Unfortunately, no consensus emerged in this session either, so there is still no clear path forward for this project. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
