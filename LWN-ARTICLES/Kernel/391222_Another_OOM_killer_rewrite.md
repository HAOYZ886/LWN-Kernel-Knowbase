---
title: Another OOM killer rewrite
url: https://lwn.net/Articles/391222/
date: "June 7, 2010"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet June 7, 2010"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 7, 2010

Nobody likes the out-of-memory (OOM) killer. Its job is to lurk out of sight until that unfortunate day when the system runs out of memory and cannot get work done; the OOM killer must then choose a process to sacrifice in the name of continued operation. It's a distasteful job, one which many think should not be necessary. But, despite the OOM killer's lack of popularity, we still keep it around; think of it as the kernel equivalent of lawyers, tax collectors, or Best Buy clerks. Every now and then, they are useful. 

The OOM killer's reputation is not helped by the fact that it is seen as often choosing the wrong victim. The fact that a running system was saved is a small consolation if that system's useful processes were killed and work was lost. Over the years, numerous developers have tried to improve the set of heuristics used by the OOM killer, with a certain amount of apparent success; complaints about poor choices are less common than they once were. Still, the OOM killer is not perfect, encouraging new rounds of developers to tilt at that particular windmill. 

For some months now, the task of improving the OOM killer has fallen to David Rientjes, who has posted several versions of his [OOM killer rewrite patch set](<http://lwn.net/Articles/391206/>). This version, he hopes, will be deemed suitable for merging into 2.6.36. It has already run the review gauntlet several times, but it's still not clear what its ultimate fate will be. 

Much of this patch set is dedicated to relatively straightforward fixes and improvements which are not especially controversial. One change opens up the kernel's final memory reserves to processes which are either exiting or are about to receive a fatal signal; that should allow them to clean up and get out of the way, freeing memory quickly. Another prevents the killing of processes which are in a separate memory allocation domain from the process which hit the OOM condition; killing those processes is unfair and unlikely to improve the situation. If the OOM condition is the result of a mempolicy-imposed constraint, only processes which might release pages on that policy's chosen nodes are considered as targets. 

Another interesting change has to do with the killing of child processes. The current OOM killer, upon picking a target for its unwelcome attention, will target one of that target's child processes if any exist. Killing the parent is likely to take out all the children anyway, so cleaning up the children - or, at least, those with their own address spaces - first may resolve the problem with less pain. The updated OOM killer does the same, but in a more targeted fashion: it attempts to pick the child which currently has the highest "badness" score, thus, hopefully, improving the chances of freeing some real memory quickly. 

Yet another change affects behavior when memory is exhausted in the low memory zone. This zone, present on 32-bit systems with 1GB or more of memory, is needed in places where the kernel must be able to keep a direct pointer to it. It is also used for DMA I/O at times. When this memory is gone, David says, killing processes is unlikely to replenish it and may cause real harm. So, instead of invoking the OOM killer, low-memory allocation requests will simply fail unless the `__GFP_NOFAIL` flag is present. 

A new heuristic which has been added is the "forkbomb penalty." If a process has a large number of children (where the default value of "large" is 1000) with less than one second of run time, it is considered to be a fork bomb. Once that happens, the scoring is changed to make that process much more likely to be chosen by the OOM killer. The "kill the worst child" policy still applies in this situation, so the immediate result is likely to be a fork bomb with 999 children instead. Even in this case, picking off the children one at a time is seen as being better than killing a potentially important server process. 

[PULL QUOTE:  The most controversial part of the patch is a complete rewrite of the `badness()` function  END QUOTE] The most controversial part of the patch is a complete rewrite of the `badness()` function which assigns a score to each process in the system. This function contains the bulk of the heuristics used to decide which process is most deserving of the OOM killer's services; over time, it has accumulated a number of tests which try to identify the process whose demise would release the greatest amount of memory while causing the least amount of user distress. 

In David's patch set, the old `badness()` heuristics are almost entirely gone. Instead, the calculation turns into a simple question of what percentage of the available memory is being used by the process. If the system as a whole is short of memory, then "available memory" is the sum of all RAM and swap space available to the system. If, instead, the OOM situation is caused by exhausting the memory allowed to a given cpuset/control group, then "available memory" is the total amount allocated to that control group. A similar calculation is made if limits imposed by a memory policy have been exceeded. In each case, the memory use of the process is deemed to be the sum of its resident set (the number of RAM pages it is using) and its swap usage. 

This calculation produces a percent-times-ten number as a result; a process which is using every byte of the memory available to it will have a score of 1000, while a process using no memory at all will get a score of zero. There are very few heuristic tweaks to this score, but the code does still subtract a small amount (30) from the score of root-owned processes on the notion that they are slightly more valuable than user-owned processes. 

One other tweak which is applied is to add the value stored in each process's `oom_score_adj` variable, which can be adjusted via `/proc`. This knob allows the adjustment of each process's attractiveness to the OOM killer in user space; setting it to -1000 will disable OOM kills entirely, while setting to +1000 is the equivalent of painting a large target on the associated process. One of the reasons why this patch is controversial is that this variable differs in name and semantics from the `oom_adj` value implemented by the current OOM killer; it is, in other words, an ABI change. David has implemented a mapping function between the two values to try to mitigate the pain; `oom_adj` is deprecated and marked for removal in 2012. 

Opposition to this change goes beyond the ABI issue, though. Understanding why is not always easy; one reviewer's [response](<https://lwn.net/Articles/391226/>) consists solely of the word ""nack"". The objections seem to relate to the way the patch replaces `badness()` wholesale rather than evolving it in a new direction, along with concerns that the new algorithm will lead to worse results. It is true that no hard evidence has been posted to justify the inclusion of this change, but getting hard evidence in this case is, well, hard. There is no simple benchmark which can quantify the OOM killer's choices. So we're left with [answers](<https://lwn.net/Articles/391227/>) like: 

I have repeatedly said that the oom killer no longer kills KDE when run on my desktop in the presence of a memory hogging task that was written specifically to oom the machine. That's a better result than the current implementation... 

Memory management patches tend to be hard to merge, and the OOM killer rewrite has certainly been no exception. In this case, it is starting to look like some sort of intervention from a higher authority will be required to get a decision made. As it happens, Andrew Morton [seems poised](<https://lwn.net/Articles/391228/>) to carry out just this sort of intervention, saying: 

The unsubstantiated "nack"s are of no use and I shall just be ignoring them and making my own decisions. If you have specific objections then let's hear them. In detail, please - don't refer to previous conversations because that's all too confusing - there is benefit in starting again. 

So, depending on what Andrew concludes, there might just be a new OOM killer in store for 2.6.36. For most users, this new feature is probably about as exciting as getting a new toilet cleaner as a birthday present. But, if it eventually helps a system of theirs survive an OOM situation in good form, they may yet come to appreciate it.
