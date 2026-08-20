---
title: "LSFMM: Lock scaling"
url: https://lwn.net/Articles/547782/
date: "April 23, 2013"
category: Scalability
author: "By Jonathan Corbet April 23, 2013 LSFMM Summit 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 23, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

Andi Kleen opened the LSFMM 2013 session on locking performance by noting that, over the years, kernel developers have worked to improve scalability by adding increasingly fine-grained locking to the kernel. Locks with small coverage help to avoid contention, but, when there _is_ contention, the resulting communication overhead wipes out the advantages that fine-grained locks are supposed to provide. We were, he said, risking the defeat of our scalability goals through excessive application of techniques designed to improve scalability. 

His main assertion was that processing single objects within a lock is not a scalable approach. What we need, he said, is interfaces that handle groups of objects in batches. Processing multiple objects under a single lock will amortize the cost of that lock over much more work, increasing the overall scalability of the system. That said, there are some challenges, starting with the choice of the optimal batch size. Developers tend to like 64, he said, but that may not be the right value. The best batch size may be hardware-dependent and needs to be tunable. 

As an example, he mentioned the per-zone LRU lock that protects the lists of pages maintained by the memory management subsystem. That lock is currently taken and released for every page processed, even when many pages need to be operated on. Taking the lock once to process a larger batch of pages would improve the scalability of the system. 

Ted Ts'o objected that the tuning of batch sizes is a problem. Lots of users will never try, and many who do will get it wrong. Automatic tuning is an obvious solution, but, there, too, there are difficulties, starting with a high-level policy question: should the system be tuned for throughput or latency? Large batches may improve throughput, but longer lock hold times can cause increased latencies — a change that some users may well object to. 

Dave Chinner pointed out that adding batching interfaces may risk creating unpredictable behavior (unknown latencies, for example) in the system; it would also increase memory use as a result of the arrays needed to track the batches. There are, he said, not that many problematic locks in the kernel. He would prefer developers to look for algorithmic changes to avoid the lock contention altogether. 

A suggestion was made that a lot of contention could be avoided by moving to message passing for communication between NUMA nodes. It is fair to say that support for this idea was minimal, though. Ted asked about which locks, in particular, would be amenable to batching; Andi responded that per-page processing in filesystem code was at the top of his list, especially in code dealing with buffered writes. There are also some locks in the networking subsystem, but the increasing use of segmentation offload means that batching is being done in the hardware. Translation lookaside buffer (TLB) flushes are also done individually, even when multiple entries need to be flushed. 

There was a general agreement that it would be a good idea to enhance the kernel to provide better information about which locks are contended, but nobody committed to actually doing that work.
