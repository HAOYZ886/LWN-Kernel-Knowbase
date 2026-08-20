---
title: Realtime adaptive locks
url: https://lwn.net/Articles/271817/
date: "March 5, 2008"
category: Realtime; Spinlocks
author: "By Jonathan Corbet March 5, 2008"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
March 5, 2008

The realtime patchset has one overriding goal: provide deterministic response times in all situations. To that end, much work has been done to eliminate places in the kernel which can be the source of excessive latencies; quite a bit of that work has been merged into the mainline over the last two years or so. One of the biggest remaining out-of-tree components is the sleeping spinlock code. Sleeping spinlocks have advantages and disadvantages. A recently posted set of patches has the potential to significantly reduce one of the biggest disadvantages of the realtime spinlock code. 

Mainline spinlocks work by repeatedly polling a lock variable until it becomes available. This busy-waiting code thus "spins" while waiting for a lock. Spinlocks are quite fast, but they can also be a source of significant latencies: a processor which is holding a lock can delay others for indefinite amounts of time. In the mainline kernel, it is also not possible to preempt a thread which holds a spinlock - another source of latencies. (See [this article](<http://lwn.net/Articles/267968/>) for a more detailed description of the mainline spinlock implementation). 

The realtime patch set addresses this problem in a couple of ways. One of those is to cause threads waiting for a contended lock to sleep rather than spin. As a result, lock contention cannot create latencies on processors which are not holding the lock. When spinning is removed, it is also possible to make code preemptible even when it holds a lock without causing deadlock problems. That allows a high-priority process to run regardless of any lower-priority processes which might currently hold locks on the current CPU. Finally, the realtime patch set has added priority awareness and priority inheritance to the locking code to ensure that the highest-priority process is always able to run. 

This is all good stuff, but there is one little disadvantage: the extra overhead imposed by the more complicated locks can reduce system throughput considerably. This is a cost that the realtime developers have been willing to pay; it is often necessary to make trade-offs between throughput and latency. Recently, though, some developers at Novell have come to the conclusion that the throughput cost of the realtime patch set need not be as severe as it currently is; the resulting [adaptive realtime locks patch](<http://lwn.net/Articles/270778/>) brings the throughput of the realtime kernel to a level much closer to that found in the mainline - at least, for some workloads. 

The core observation encapsulated in this patch set is that hold times for spinlocks tend to be quite short, especially in the realtime kernel. So the cost of putting a waiting thread to sleep may well exceed the cost of simply busy-waiting until the lock becomes free. So adaptive locks behave more like their mainline counterpart and simply spin until the lock becomes available. There are some twists, though, which are necessitated by the realtime system: 

  * The spinning cannot go on forever, since it may cause unacceptable latencies elsewhere in the system. So an adaptive lock will only spin up to a configurable number of times (the default is 10,000) before giving up and going to sleep. 

  * Since lock holders are preemptible in the realtime kernel, it is possible that the thread which currently holds the lock was previously running on the same CPU as the process trying to acquire the lock. In that situation, spinning for the lock is clearly a bad thing to do. In the absence of a loop counter, it would be a hard deadlock situation; with the counter, it would just be an unnecessary delay. Either way, the result is undesirable, so, if the lock owner is running on the same processor, the thread waiting for the lock simply goes to sleep. 

  * If the lock owner is, instead, itself sleeping while waiting for something, there is little point in having another thread stay awake in the hope that the owner will release the lock soon. So, in this case too, a thread contending for a lock will simply go to sleep rather than spin. 

One other throughput improvement is obtained by changing the lock-stealing code. Locks in the realtime system are normally fair, in that threads waiting for a lock will get it in first-come-first-served order. A higher-priority process will jump the queue, however, and "steal" the lock from lower-priority processes which have been waiting for longer. The adaptive locks patch tweaks this algorithm by allowing a running process to steal a lock from another, equal-priority process which is sleeping. This change adds some unfairness to the locking code, but it allows the system to avoid a context switch and keep a running, cache-warm process going. 

[Some benchmark results [PDF]](<ftp://ftp.novell.com/dev/ghaskins/adaptive-locks.pdf>) have been posted. On the test system, the dbench benchmark runs at about 1500 MB/s on a stock 2.6.24 system, but at just under 170 MB/s on a system with the realtime patches applied. The adaptive lock patch raises that number back to over 700 MB/s - still far from a mainline system, but much better than before. The improvement in hackbench results is even better, while the change in the all-important "build the kernel" benchmark is small (but still positive). A fundamental patch like this will require quite a bit of review and testing before it might be accepted. But the initial results suggest that adaptive locks might be a big win for the realtime patch set.
