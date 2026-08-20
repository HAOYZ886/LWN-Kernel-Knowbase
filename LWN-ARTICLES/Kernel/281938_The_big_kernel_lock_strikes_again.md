---
title: The big kernel lock strikes again
url: https://lwn.net/Articles/281938/
date: "May 13, 2008"
category: Big kernel lock; Latency; Semaphores; lock kernel
author: "By Jonathan Corbet May 13, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
May 13, 2008

When Alan Cox first made Linux work on multiprocessor systems, he added a primitive known as the big kernel lock (or BKL). This lock, originally, ensured that only one processor could be running kernel code at any given time. Over the years, the role of the BKL has diminished as increasingly fine-grained locking - along with lock-free algorithms - have been implemented throughout the kernel. Getting rid of the BKL entirely has been on the list of things to do for some time, but progress in that direction has been slow in recent years. A recent performance regression tied to the BKL might give some new urgency to that task, though; it also shows how subtle algorithmic changes can make a big difference. 

The AIM benchmark attempts to measure system throughput by running a large number of tasks (perhaps thousands of them), each of which is exercising some part of the kernel. Yanmin Zhang [reported](<https://lwn.net/Articles/281958/>) that his AIM results got about 40% worse under the 2.6.26-rc1 kernel. He took the trouble to bisect the problem; the guilty patch turned out to be the [generic semaphores](<http://lwn.net/Articles/273731/>) code. Reverting that patch made the performance regression go away - at the cost of restoring over 7,000 lines of old, unlamented code. The thought of bringing back the previous semaphore implementation was enough to inspire a few people to look more deeply at the problem. 

It did not take too long to narrow the focus to the BKL, which was [converted to a semaphore](<http://lwn.net/Articles/102253/>) a few years ago. That part of the process was easy - there aren't a whole lot of other semaphores left in the kernel, especially in performance-critical places. But the BKL stubbornly remains in a number of core places, including the `fcntl()` system call, a number of `ioctl()` implementations, the TTY code, and `open()` for char devices. That's enough for a badly-performing BKL to create larger problems, especially when running VFS-heavy benchmarks with a lot of contention. 

Ingo Molnar [tracked down the problem](<https://lwn.net/Articles/281961/>) in the new semaphore code. In short: the new semaphore code is too fair for its own good. When a semaphore is released, and there is another thread waiting for it, the semaphore is handed over to the new thread (which is then made runnable) at that time. This approach ensures that threads obtain the semaphore in something close to the order in which they asked for it. 

The problem is that fairness can be expensive. The thread waiting for the semaphore may be on another processor, its cache could be cold, and it might be at a low enough priority that it will not even begin running for some time. Meanwhile, another thread may request the semaphore, but it will get put at the end of the queue behind the new owner, which may not be running yet. The result is a certain amount of dead time where no _running_ thread holds the semaphore. And, in fact, Yanmin's experience with the AIM benchmark showed this: his system was running idle almost 50% of the time. 

The solution is to bring in a technique from the older semaphore code: lock stealing. If a thread tries to acquire a semaphore, and that semaphore is available, that thread gets it regardless of whether a different thread is patiently waiting in the queue. Or, in other words, the thread at the head of the queue only gets the semaphore once it starts running and actually claims it; if it's too slow, somebody else might get there first. In human interactions, this sort of behavior is considered impolite (in some cultures, at least), though it is far from unknown. In a multiprocessor computer, though, it makes the difference between acceptable and unacceptable performance - even a thread which gets its lock stolen will benefit in the long run. 

Interestingly, the patch which implements this change was merged into the mainline, then reverted before 2.6.26-rc2 came out. The initial reason for the revert was that the patch broke semaphores in other situations; for some usage patterns, the semaphore code could fail to wake a thread when the semaphore became available. This bug could certainly have been fixed, but it appears that things will not go that way - there is a bit more going on here. 

What is happening instead is that Linus has committed a patch which simply turns the BKL into a spinlock. By shorting out the semaphore code entirely, this patch fixes the AIM regression while leaving the slow (but fair) semaphore code in place. This change also makes the BKL non-preemptible, which will not be entirely good news for those who are concerned with latency issues - especially the real time tree. 

The reasoning behind this course of action would appear to be this: both semaphores and the BKL are old, deprecated mechanisms which are slated for minimization (semaphores) or outright removal (BKL) in the near future. Given that, it is not worth adding more complexity back into the semaphore code, which was dramatically simplified for 2.6.26. And, it seems, Linus [is happy with a sub-optimal BKL](<https://lwn.net/Articles/281971/>): 

Quite frankly, maybe we _need_ to have a bad BKL for those to ever get fixed. As it was, people worked on trying to make the BKL behave better, and it was a failure. Rather than spend the effort on trying to make it work better (at a horrible cost), why not just say "Hell no - if you have issues with it, you need to work with people to get rid of the BKL rather than cluge around it". 

So the end result of all this may be a reinvigoration of the effort to remove the big kernel lock from the kernel. It still is not something which is likely to happen over the next few kernel releases: there is a lot of code which can subtly depend on BKL semantics, and there is no way to be sure that it is safe without auditing it in detail. And that is not a small job. Alan Cox has been reworking the TTY code for some time, but he has some ground to cover yet - and the TTY code is only part of the problem. So the BKL will probably be with us for a while yet.
