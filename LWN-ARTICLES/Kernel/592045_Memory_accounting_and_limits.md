---
title: Memory accounting and limits
url: https://lwn.net/Articles/592045/
date: "March 26, 2014"
category: Memory management
author: "By Jonathan Corbet March 26, 2014 2014 LSFMM Summit"
---

By **Jonathan Corbet**  
March 26, 2014

* * *

[2014 LSFMM Summit](<https://lwn.net/Articles/LSFMM2014/>)

Two separate sessions in the memory management track of the 2014 Linux Storage, Filesystem, and Memory Management Summit looked at memory accounting and the application of limits to memory usage. One would think that this old problem would have been solved long ago, but it is clear that there are still a number of open issues in this area. 

#### Low limits

At the 2013 Summit, Michal Hocko [tried to convince developers](<https://lwn.net/Articles/548189/>) that a change in how the "soft" limit in memory control groups ("memcgs") are implemented was needed. He was not successful in that attempt, so, this year, he came back with a variation of that approach: rather than change soft limits, he would like to add a new limit to memcgs called the "low limit." 

A soft limit is meant to provide an upper bound on memory consumption when the system is under memory pressure. If there is plenty of memory available, a memcg can consume more than its soft limit would allow, but, when pressure hits, the reclaim code will step in and the memcg's use will be cut back quickly to the soft [![\[Michal Hocko\]](https://static.lwn.net/images/conf/2014/lsfmm/MichalHocko-sm.jpg)](<https://lwn.net/Articles/592108/>) limit. If the memory pressure persists, processes in memcgs may be cut back even further, well below the soft limit set by the memcg. But sometimes users don't want certain memcgs to go below a minimum amount of memory even when the memory pressure is severe. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

That is the purpose of the low limit. If this limit is set on a memcg, the memory management subsystem will not reduce that memcg's usage below the limit even if the system is desperately short of memory. The low limit is meant to be a sort of guarantee; the system takes it seriously enough that it will go into a full out-of-memory condition before it will reduce a memcg below its low limit. 

There were a couple of questions that resulted from this presentation. Peter Zijlstra went back to the idea of using the soft limit as a guarantee instead. Since nobody seems to like how soft limits are implemented now, why not just change things? Part of the problem is that the current default soft limit is "unlimited"; using the soft limit as a guarantee would require changing that default to zero. Whether that change (which constitutes an ABI change) would affect users is unclear; as Peter put it, anybody who is actually using soft limits is already changing that value anyway. But Michal, who fought this battle for a while, is nervous about changing that interface now, and he is not the only one. 

Other developers questioned the wisdom of setting up a limit mechanism that is designed to push the system into out-of-memory situations. They don't feel that a minimal amount of memory can ever be guaranteed to a memcg, since the total amount of available memory cannot be guaranteed. But, in the end, most seem willing to let Michal try; if users break their systems with it, they get to keep all of the pieces. 

But, in contrast to last year's discussion, Michal may well be pushed back toward using the soft limit rather than adding a new one. Some developers don't want to add yet another limit. There is also universal disdain for the current soft limit code, which, it is said, should not be viewed shortly after meals by developers with sensitive stomachs. Changing the way the limits work would enable the removal of much of that code. If soft limits are used, a simple "`oom`" Boolean flag could be added to allow users to request the "low limit" behavior; this flag would not be set by default. If the current view doesn't change, that is the form that the next version of this patch set will take. 

#### Memory pinning

Peter Zijlstra got up to talk about situations where drivers need to allocate "pinned" pages — pages in a process's address space that cannot be swapped out or even migrated between processors. Pinning is useful for buffers used in RDMA conversations, with the perf events subsystem, and for video frame buffers, among other things. Once upon a time, Peter said, pinned pages were treated much like pages locked into memory with `mlock()` for accounting purposes. Either type of page would be accounted against the mlock limit, placing an upper bound on the total amount of memory a process could lock down. 

More recently, he said, the accounting changed so that pinned pages are counted separately from locked pages. That essentially doubled the amount of memory a process could lock down. On some systems, that meant that processes were now able to push the system into an out-of-memory condition, which is not desirable. So Peter would like to revert the accounting back to the way it was before. 

Andrew Morton replied that this could be hard. The kernel has been, for better or worse, changed to be more permissive; going back now could break things for other users. In the end, that view may carry, though no real conclusion was reached in the session. 

One reason that Peter is looking at this functionality is that developers in the realtime community are figuring out that `mlock()` doesn't quite give them the guarantees they would like to have. Locking a page into memory guarantees that it will not be swapped out, but it still gives the kernel some freedom; in particular, the kernel is free to migrate a locked page between locations in RAM. Migration can cause delays and soft page faults for realtime applications, which is not welcomed by realtime developers. 

As it happens, the kernel does not currently migrate locked pages, but the memory management developers reserve the right to do so in the future. So Peter is looking at adding a new set of system calls, `mpin()` and `munpin()`, that would fully pin pages in memory. When those calls go in, it would be nice to have a clear view of how the accounting will work. At the moment, it appears that pinned pages will go into a different accounting bin than locked pages. 

[Your editor would like to thank the Linux Foundation for supporting his travel to the Summit.]
