---
title: "Preventing overly-optimistic spinning"
url: https://lwn.net/Articles/401911/
date: "August 25, 2010"
category: "Locking mechanisms-Mutexes"
author: "By Jonathan Corbet August 25, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
August 25, 2010

A kernel mutex is a sleeping lock; a thread which loses in contention for a specific mutex will be blocked until that mutex becomes available. At least, that's what the documentation says; the reality is a bit more complicated. Experience has shown that throughput can sometimes be improved if processes waiting for a lock do not go to sleep immediately. In particular, if (1) a thread finds a mutex to be unavailable, and (2) the holder of the mutex is currently running, that thread will spin until the mutex becomes available or the holder blocks. That "optimistic" spinning allows the transfer of the mutex without going through a sleep/wakeup cycle, and, importantly, it gives the mutex to a running (and, thus, cache-hot) thread. The result is an unfair, but better-performing mutex implementation. 

Except that, as it turns out, it doesn't always perform better. While doing some testing on a 64-core system, Tim Chen noticed a problem: multiple threads can be waiting for the same mutex at any given time. Once the mutex becomes available, only one of this spinning threads will obtain it; the others will continue to spin, contending for the lock. In general, optimism can be good, but excessive optimism can be harmful if it leads to continued behavior which does not yield useful results. That would appear to be the case here. 

Tim's response was [a patch](<https://lwn.net/Articles/401912/>) changing the optimistic spinning implementation slightly. There is now an additional check in the loop to see if the owner of the mutex has changed. If the ownership of a mutex changes while a thread is spinning, waiting for it, that means that it was released and somebody else grabbed it first. In other words, there is heavy contention and multiple CPUs are spinning in a race that only one of them can win. In such cases, it makes sense to just go to sleep and wait until things calm down a bit. 

Various benchmark results showed significant performance improvements in heavily-contended situations. That was enough to get the patch merged for 2.6.36-rc2.
