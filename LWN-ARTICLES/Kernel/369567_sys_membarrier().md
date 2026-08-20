---
title: sys_membarrier()
url: https://lwn.net/Articles/369567/
date: "January 13, 2010"
category: "Read-copy-update; membarrier"
author: "By Jonathan Corbet January 13, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 13, 2010

Mathieu Desnoyers is the longtime developer of the [LTTng](<http://ltt.polymtl.ca/>) tracing toolkit. A current project of his is to provide for fast tracing of multithreaded user-space applications; that, in turn, requires a fast, multithreaded tracing utility. Tracing is controlled through a shared memory area; to make that control as fast as possible, Mathieu would like to use the read-copy-update (RCU) algorithm. That, in turn, means that he has been working on porting RCU - a kernel-only technology - to user space. In the process, he has run into some interesting challenges. 

As with the kernel version, user-space RCU works by deferring the cleanup of in-memory objects until it is known that no more references to those objects can exist. The implementation must be done differently, though, since user-space code is unable to run in the same atomic mode used by RCU in the kernel. So, in user space, a call to `rcu_read_lock()` sets a variable in shared memory indicating that the thread is in an RCU critical section. Within that critical section, it's safe for the thread to access RCU-protected variables. 

...at least, it's safe as long as nobody reorders operations in a way that causes an access to happen to an RCU-protected variable before the effects of `rcu_read_lock()` are visible to other CPUs. That kind of reordering can indeed happen, at both the compiler and CPU levels, so it's a problem which must be addressed. Compile-time reordering is relatively easy to avoid, but runtime reordering in the CPU requires the issuing of a memory barrier instruction. And, indeed, user-space RCU can be made to work by putting memory barriers into the `rcu_read_lock()` call. 

The problem with that solution is that memory barriers slow things down significantly. Even worse, they slow down the fast path for a case - a change to an RCU-protected variable - which happens rarely. So Mathieu would like to get rid of that barrier. To that end, he coded up a solution which sends a signal to every thread when an RCU-protected variable is about to be changed, forcing each thread to execute a memory barrier at that time. This solution does speed things up, believe it or not, but signals are almost never the optimal solution to any problem. Mathieu would like to do something better. 

His "something better" turned out to be [a simple system call](<http://lwn.net/Articles/369018/>): 

```
void membarrier();
```

The initial implementation would simply send an inter-processor interrupt to every CPU in the system; the receiving CPUs would respond by executing an explicit memory barrier instruction. The solution worked, but it ran into a couple of objections in review: 

  * By allowing a user-space program to force interrupts to all processors on the system, `membarrier()` presented an easy way to create denial-of-service attacks on the system. 

  * The system call interrupted every processor on the system. Interrupting processors running different applications is a small but useless waste. The problem gets a little worse if some of those CPUs are running realtime tasks, which, presumably, would not welcome the forced addition of a bit of latency into their world. It would even interrupt processors which were currently sleeping - a useless exercise which would also increase energy use. 

What followed was a long discussion on how to optimize the patch, whether an explicit memory barrier is needed even after the CPU has taken an inter-processor interrupt (the safe answer appears to be "yes"), and so on. All told, an impressive amount of effort has gone into the optimization of a small patch which is, at its core, implementing the slow path which should be rarely executed. 

Current status, as of this writing, is that Mathieu has posted [a new version of the patch](<https://lwn.net/Articles/369640/>) with a number of changes. The first of those is the addition of an integer "`expedited`" parameter. If this value is zero, the system call simply calls `synchronize_sched()` and returns; this is the cheapest way of getting the needed functionality, but it comes at the cost of a latency of some milliseconds for the caller. It seems clear that Mathieu expects the "expedited" mode to be used most of the time. 

For an expedited barrier, the system call will look at every CPU in the system, building a mask of those which are running in the same address space as the caller; those CPUs will then receive the inter-processor interrupt asking them to execute a memory barrier instruction. It's a rather more complicated implementation, but, since it only interrupts processors which are running the calling application, the denial of service, performance, and energy use concerns are no longer relevant. One assumes that the patch is getting close to its final form, but it's hard to say for sure: sometimes it's the smallest and simplest patches which are scrutinized the most.
