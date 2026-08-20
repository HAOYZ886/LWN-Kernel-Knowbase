---
title: Background writeback
url: https://lwn.net/Articles/685894/
date: "May 4, 2016"
category: "Memory management-Writeback"
author: "By Jake Edge May 4, 2016 LSFMM 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 4, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

The problems with [background writeback](<https://lwn.net/Articles/682582/>) in Linux have been known for quite some time. Recently, there has been an effort to apply what was learned by network developers [solving the bufferbloat problem](<https://lwn.net/Articles/616241/>) to the block layer. Jens Axboe led a filesystem and storage track session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit to discuss this work. 

The basic problem is that flushing block data from memory to storage (writeback) can flood the device queues to the point where any other reads and [ ![\[Jens Axboe\]](https://static.lwn.net/images/2016/lsf-axboe-sm.jpg) ](<https://lwn.net/Articles/685988/>) writes experience high latency. He has posted several versions of [a patch set](<https://lwn.net/Articles/685236/>) to address the problem and believes it is getting close to its final form. There are fewer tunables and it all just basically works, he said. 

The queues are managed on the device side in ways that are "very loosely based on [CoDel](<https://en.wikipedia.org/wiki/CoDel>)" from the networking code. The queues will be monitored and write requests will be throttled when the queues get too large. He thought about dropping writes instead (as CoDel does with network packets), but decided "people would be unhappy" with that approach. 

The problem is largely solved at this point. Both read and write latencies are improved, but there is still some tweaking needed to make it work better. The algorithm is such that if the device is fast enough, it "just stays out of the way". It also narrows in on the right queue size quickly and if there are no reads contending for the queues, it "does nothing at all". He did note that he had not yet run the "crazy Chinner [test case](<https://lwn.net/Articles/683353/>)" again. 

Ted Ts'o asked about the interaction with the I/O controller for control groups that is trying to do proportional I/O. Axboe said he was not particularly concerned about that. Controllers for each control group will need to be aware of each other, but it should all "probably be fine". 

David Howells asked about writeback that is going to multiple devices. Axboe said that still needs work. Someone else asked about background reads, which Axboe said could be added. Nothing is inherently blocking that, but the work still needs to be done.
