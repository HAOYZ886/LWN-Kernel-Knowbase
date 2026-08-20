---
title: "NUMA-aware qspinlocks"
url: https://lwn.net/Articles/852138/
date: "April 12, 2021"
category: NUMA; Spinlocks
author: "By Jonathan Corbet April 12, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 12, 2021

While some parts of the core kernel reached a relatively stable "done" state years ago, others never really seem to be finished. One of the latter variety is undoubtedly the kernel's implementation of spinlocks, which arbitrate access to data at the lowest levels of the kernel. Lock performance can have a significant effect on the performance of the system as a whole, so optimization work can pay back big dividends. Lest one think that this work is finally done, the [NUMA-aware qspinlock patch set](<https://lwn.net/ml/linux-kernel/20210401153156.1165900-1-alex.kogan@oracle.com/>) shows how some more performance can be squeezed out of the kernel's spinlock implementation. 

In its simplest form, a spinlock is a single word in memory, initially set to one. Any CPU wishing to acquire the lock will perform an atomic decrement-and-test operation; if the result is zero, the lock has been successfully taken. Otherwise the CPU will increment the value, then "spin" in tight loop until the operation succeeds. The kernel has long since left this sort of implementation behind, though, for a number of reasons, including performance. All those atomic operations on the lock word cause its cache line to be bounced around the system, slowing things considerably even if contention for the lock is light. 

The current "qspinlock" implementation is based on MCS locks, which implement a queue of CPUs waiting for the lock as a simple linked list. Normally, linked lists are just the sort of data structure that one wants to avoid when cache efficiency is a concern, but nobody ever has to traverse this list. Instead, each CPU will spin on its own entry in the list, and only reach into the next entry to release the lock. See [this article](<https://lwn.net/Articles/590243/>) for a more complete description, complete with cheesy diagrams, of how MCS locks work. 

#### MCS locks on NUMA systems

MCS locks seem nearly optimal; each CPU focuses on its own queue entry, so cache-line bouncing between processors is nearly eliminated. They are also fair; the queue of waiters ensures that no CPU is starved of access. But it seems that there is a way to do better, at least on non-uniform memory-access (NUMA) systems. Such machines are made up of multiple nodes, each of which contains some number of CPUs; memory attached to a CPU's node will be faster to access than memory attached to a remote node. Access to cached memory is (relatively) fast, of course, regardless of the node that memory is attached to, but moving cache lines between nodes is expensive, even more expensive than bouncing cache lines between CPUs on the same node. Thus, minimizing cache-line movement between NUMA nodes will be good for performance. 

If a spinlock is released by a CPU on one node and subsequently acquired by a CPU on a different node, its cache line will have to move between the nodes. If, instead, a contended spinlock could be passed to another CPU on the same node, that expense will be avoided. That alone can make a difference, but it's worth remembering that spinlocks protect data structures. Two processors contending for a given lock are quite likely to be trying to access the same data, so moving the lock between nodes will also cause the cache lines for the protected data to move. For heavily contended data structures, the resulting slowdown can hurt. 

The NUMA-aware qspinlock attempts to keep locks from bouncing between NUMA nodes by handing them off to another CPU on the same node whenever possible. To do this, the queue of CPUs waiting for the lock is split into two — a primary and secondary queue. If a CPU finds the lock unavailable, it will add itself to the primary queue and wait as usual. When a CPU gets to the head of the queue, though, it will look at the next CPU in line; if that next CPU is on a different NUMA node, it will be shunted over to the secondary queue. 

In this way, the waiting CPUs will eventually be sorted into two queues, one of which (the primary queue) consists only of CPUs on the same node as the current owner of the lock, and one (the secondary) which contains all the rest. When a CPU releases the lock it will hand it to the next CPU in the primary queue, thus keeping the lock on the same NUMA node. The lock will only move to another node once the primary queue empties out, at which point the secondary queue will be moved to the primary and the process starts all over again. 

#### Tweaks and benchmarks

There is an obvious pitfall with a scheme like this: if the lock is heavily contended, the primary queue may never empty out and the other nodes in the system will be starved for the lock. The solution to this problem is to make a note of the time when the first CPU was moved to the secondary queue. If the primary queue does not empty out for 10ms (by default), the entire secondary queue will be promoted to the head of the primary queue, thus forcing the lock to move to another node. The timeout can be changed (within a range of 1-100ms) with the `numa_spinlock_threshold` command-line parameter. 

One optimization that has been added is called "shuffle reduction". If the lock is not all that heavily contended, the extra work of maintaining the secondary queue does not really buy anything. To mitigate this extra cost, the code uses a pseudo-random number generator to only try to create the secondary queue one time out of every 128 lock acquisitions. If the lock gets busy, that will happen relatively often, after which the secondary queue will be maintained until the primary queue empties again (or the above-mentioned timeout occurs). 

Finally, the code exempts CPUs running in interrupt (or non-maskable interrupt) mode, and those running realtime tasks, from being pushed to the secondary queue. That allows these CPUs, which presumably have a higher priority, to acquire the lock relatively quickly even if they are running on the wrong NUMA node. 

A number of benchmark results are included with the patch set. For lightly contended locks the performance benefits of NUMA awareness are relatively modest. As the number of contending threads grows, though, the speedup does as well, approaching a factor of two for ridiculously heavily contended loads. 

This patch set has been through 14 revisions since it was [first posted](<https://lwn.net/ml/linux-kernel/20190131030136.56999-1-alex.kogan%40oracle.com/>) in January 2019. It has evolved quite a bit over that time as comments were raised and addressed; it would appear to be approaching a sort of steady state where it is getting close to being ready to merge. Given that this work has been pending for over two years already, though, and given that it makes significant changes to one of the kernel's fundamental synchronization primitives, it would not be surprising if it took a little longer yet before it hits the mainline. 

[This paper](<https://arxiv.org/abs/1810.05600>) describes the NUMA-aware qspinlock algorithm in more detail, though the details of the implementation have diverged somewhat from what is described there.
