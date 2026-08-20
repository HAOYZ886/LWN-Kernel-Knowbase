---
title: "Out-of-lining spinlocks"
url: https://lwn.net/Articles/97537/
date: "August 11, 2004"
category: Spinlocks
author: 
---

Spinlocks, as the core kernel synchronization primitive, are highly performance critical. They are implemented differently on each architecture, by way of some carefully-crafted assembly code, so that not one extra cycle is spent there, especially when the lock is not contended. They are also implemented as inline assembly, so that no function calls get in the way of that fast path through. 

Recently, however, Zwane Mwaikambo has pulled [a patch](<https://lwn.net/Articles/97093/>) out of the -tiny tree which moves spinlocks into normal, out-of-line functions - at least, on the x86 and x86-64 architectures. The reason for doing this is to shrink the kernel; there are a _lot_ of spinlock calls in the kernel, and the inline code gets replicated for every one of them. Moving the spinlock code out of line gets rid of that duplication, and shrinks the kernel text size by 50KB or so. 

Zwane posted some benchmarks showing that there are no performance regressions. In fact, on some hardware, the improved cache utilization brought about by pulling together the spinlock code can actually improve performance by a slight amount. 

The patch comes with a configuration option allowing the spinlock code to be built in either mode. Given that moving the code out of line seems to be a win, some have wondered if things shouldn't always be done that way. Linus [pointed out](<https://lwn.net/Articles/97538/>) one advantage to the inline code: it makes the sources of lock contention very clear in kernel profiles. With out-of-line spinlocks, all a profile will show is that a lot of time was spent waiting for locks; with the code inline, the function which is actually waiting for the lock shows up instead. So out-of-line locks may be best for production kernels, but developers may want to keep them inline.
