---
title: "LSFMM: Reducing io_submit() latency"
url: https://lwn.net/Articles/548352/
date: "May 1, 2013"
category: Asynchronous IO; Syslets
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

At the 2013 LSFMM Summit, Ankit Jain led a discussion of ways to reduce the latency in the `io_submit()` system call which is used to submit asynchronous I/O operations to the kernel. He doesn't have a solution to the problem that `io_submit()` can sometimes sleep especially when allocating new blocks, so he was interested in hearing from attendees on what kinds of solutions might be available. 

Jain described a "naïve" approach he took that offloaded the processing of all `iocbs` (I/O control blocks that are used to track asynchronous I/O requests) to workqueues. That way, the workqueue thread would block, but `io_submit()` would not. That didn't work very well for modern flash disks. It also depends on the submitter's context, and may need to be integrated with control groups. That led him to look at other solutions such as filesystem-specific approaches, [syslets](<https://lwn.net/Articles/261473/>), or [fibrils](<https://lwn.net/Articles/219954/>). Jain said that fibrils were a "neat solution" and he was not sure why they were not accepted. 

Zach Brown cautioned that there is "no quick fix" for handling this problem. There are "no magic workqueue tricks" that will solve the problem and any path to a solution is "incredibly painful". Beyond that, though, there is no consensus among kernel developers as to the right direction. Fibrils have pain points; syslets have a different set, as do thread pools. Essentially, Jain is asking for a "consensus that hasn't existed for five years". Al Viro suggested that five years was far too low; that the consensus had eluded the kernel community for much longer than that. 

Brown said that "arguably" the filesystem-specific solution is the least work. For ext4, allocation data structures can be pinned in memory so that they don't need to be read at `io_submit()` time. Ted Ts'o said that solving the general case problem was so painful that it may really make no sense to do so. He has a solution that will eliminate waits in block allocation as long as the file is not being extended. There will be an occasional sleep in `kmalloc()`, but if that can be lived with, his ext4-specific solution may be enough. 

Solving the general problem will be very expensive in terms of development cost, Ts'o said. But it may also cause a lot of "spillover pain" to the community, which may make it difficult to get it merged. Lots of people have looked at the problem over the years and "decided to hack it" for their specific use case, he said. 

But David Howells said that he is looking at reworking the Linux I/O subsystem into an event-based framework. Filesystem I/O would be broken up into individual state machines, which would make asynchronous I/O simpler and "completely async" rather than "semi-async". There is quite a bit of work to do, he said, but he thinks he can make it work and asked others who were interested to talk to him later. 

Dave Chinner stepped back to say that it was impossible to solve the problem of knowing when an allocation will block. There are locks taken at multiple levels throughout the I/O stack, and rolling back a transaction because it will block could also require locks, which could cause blocking. Overall, there are complex interactions that are specific to the filesystem in question, so you would end up with "ten different solutions". 

Jain suggested that fibrils would avoid those problems, but Brown said it would just shift the problem to "scarier things". Sharing a `task` struct between multiple threads requires a code audit to ensure there aren't concurrent access issues. But even if that were done, it would require eternal vigilance whenever new code is added that touches those structures, Ts'o said. Fibrils are a fragile solution. 

Is this really a problem that needs to be solved, James Bottomley asked, since no one has been bitten by it hard enough to provide resources to solve it? Boaz Harrosh said that threads could handle most asynchronous work as long as you don't need "10,000 operations in parallel" because that would require too many threads. In general, the complaints come from database companies about the latency of asynchronous I/O, Jan Kara said. 

In the end, the database companies just want a way to submit writes to many disjoint blocks, Brown said. If there were an interface like `writev()` that also did file positioning in addition to scatter-gather, that would probably be good enough. The database companies just want to be able to kick off many concurrent asynchronous I/O operations, so a new system call to do so would likely keep them happy. 

There was a bit of talk about various patches that might or might not help the problem, but the overall sense was that there is no easy (or even hard) solution to Jain's problem. 

[ Thanks are due to Elena Zannoni, whose detailed notes were a nice supplement to my own. ]
