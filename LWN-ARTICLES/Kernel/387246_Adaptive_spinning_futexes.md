---
title: Adaptive spinning futexes
url: https://lwn.net/Articles/387246/
date: "May 11, 2010"
category: Futex
author: "By Jonathan Corbet May 11, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 11, 2010

As a general rule, a well-written program should, when it needs a resource currently owned by another program, step aside and allow other work to proceed until that resource becomes available. When it comes to low-level synchronization primitives, though, this rule does not always hold. Better overall system performance can often be achieved if a program busy-waits rather than sleeping. If the wait is short, the performance benefits that come from giving the resource to an already-running, cache-hot process outweigh the cost of the busy wait. 

The best-supported (by the kernel) user-space synchronization primitive is the futex. Darren Hart has been working on [a patch series](<http://lwn.net/Articles/386536/>) intended to bring adaptive spinning to futexes in an attempt to improve the performance of multi-threaded applications. These patches, while still marked as "not ready for inclusion," have evolved considerably over time. 

The core idea is simple: if a process attempts to acquire a futex which is already owned by another, it will spin in an acquisition loop until the holding process either releases the futex or is scheduled out. If all goes well, the new process will be able to grab the futex quickly and get on with its work in the most efficient way. In practice, adaptive spinning generally outperforms regular futexes, but only occasionally does better than the highly tweaked, assembly-coded adaptive spinning mutex code used by the pthreads library. 

Adaptive spinning requires that the kernel know which process currently owns the futex; that is a minor problem because the current futex operations do not provide that information. So a new locking operation is required in situations where adaptive spinning is to be used. 

There is an alternative approach which has been recommended by some developers: do the spinning in user space rather than in the kernel. User-space spinning might just be faster, but it's trickier, because it's harder for user space to know whether the current holder of a futex is executing or not. Providing the requisite information will require the design of a special (and fast) API - work which has not yet been done.
