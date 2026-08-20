---
title: A surprise with mutexes and reference counts
url: https://lwn.net/Articles/575460/
date: "December 4, 2013"
category: "Locking mechanisms-Mutexes; Race conditions; Reference counting"
author: "By Jonathan Corbet December 4, 2013"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 4, 2013

A kernel crash first [reported](<https://lwn.net/Articles/575474/>) in July for the 3.10 kernel has finally been diagnosed. In the process, some suspect locking in the pipe code has been fixed, but there is a more interesting outcome as well: it appears that an "obviously correct" reference count code pattern is not so correct after all. Understanding why requires looking at the details of how mutexes are implemented in the Linux kernel. 

Conceptually, a kernel mutex is a simple sleeping lock. Once a lock is obtained with `mutex_lock()`, the owner has exclusive access to the protected resources. Any other thread attempting to acquire the same lock will block until the lock becomes free. In practice, the implementation is a little more complex than that. As was first [demonstrated](<https://lwn.net/Articles/313682/>) in the Btrfs code, performance can be improved if a thread that encounters an unavailable lock actively spins for a brief period (in the hope that the lock will be quickly released) before going to sleep. A running thread that is able to quickly grab a mutex in this manner is likely to be cache-hot, so this type of optimistic spinning tends to improve the overall throughput of the system. Needless to say, the period of spinning should be relatively short; it also will not happen at all if another thread is already spinning or if the owner of the mutex is not actually running. 

With that background in place, consider a piece of code like the following which manipulates a structure called "`s`"; that structure is protected by a mutex embedded within it: 

```
int free = 0;
    
        mutex_lock(&s->lock);
        if (--s->refcount == 0)
            free = 1
        mutex_unlock(&s->lock);
        if (free)
    	kfree(s);
```

This code looks like it should work fine; the structure is only freed if it is known that no more references to it exist. But, as Linus [figured out](<https://lwn.net/Articles/575477/>) while pondering the old bug report, the above code is not correct at all with the current mutex implementation. 

The core of the mutex data structure includes an atomic counter (`count`) and a spinlock (`wait_lock`). When the lock is free, `count` has a value of one. A call to `mutex_lock()` on an available lock will simply decrement `count` to zero and continue; that is the fast path that, with luck, will be executed most of the time. Should the lock be taken, though, `count` will be set to a negative number (to indicate that somebody else wants the lock), and the frustrated caller will possibly spin, waiting for `count` to be set to one once again. 

When the call to `mutex_unlock()` comes, the fast path (executed when `count` is zero) is, once again, simple: `count` is incremented back to a value of one. Should `count` be negative, though, the slow-path code must be run to ensure that waiting threads know that the lock has become available. In a simplified form, this code does the following: 

```
spin_lock(&lock->wait_lock);
        atomic_set(&lock->count, 1);
        wake_up_process(/* the first waiting thread */);
        spin_unlock(&lock->wait_lock);
```

The problem can now be seen: the thread which is spinning, waiting for the lock to be free, will break out of its loop once it sees the effect of the `atomic_set()` call above. So, while the original lock holder is calling `wake_up_process()` to wake somebody who is waiting for the lock, a thread on another CPU is already proceeding in the knowledge that it owns the same lock. If that thread quickly frees the data structure containing the lock, the final `spin_unlock()` call will make changes to memory that has already been freed and, possibly, allocated to another user. It is an uncommon race to hit, but, as the original bug report shows, it can occasionally happen. 

All this led Linus to conclude: 

In other words, it's unsafe to protect reference counts inside objects with anything but spinlocks and/or atomic refcounts. Or you have to have the lock *outside* the object you're protecting (which is often what you want for other reasons anyway, notably lookup). 

There is an obvious question that immediately arises from this conclusion: how many other places in the kernel might be affected by this kind of bug? Mutexes and reference counts are both heavily used in the kernel; there must certainly be other places that use the two of them together in an unsafe manner (though Linus [is doubtful](<https://lwn.net/Articles/575494/>) that they exist). Needless to say, it would be nice to fix that code; the real question is how that might be done. One option is to audit all mutex-using code to find problematic usage, but that would be a long and unpleasant task — one that is unlikely to ever be completed in a comprehensive manner. 

The alternative — fixing mutexes to eliminate the failure mode described above — seems easier and more robust in the long term. Ingo Molnar [suggested](<https://lwn.net/Articles/575492/>) a couple of ways in which that could be done, both based on the understanding that the use of both `count` and `wait_lock` to protect parts of the lock is at the root of the problem. The first suggestion was to eliminate `count` and turn `wait_lock` into a sort of special, tri-state spinlock; the other, instead, is to use `count` to control all access to the lock. Either approach, it seems, has the potential to reduce the size of the mutex structure and reduce architecture-specific code along with fixing the problem. 

As of this writing, no patches have been posted. It would be surprising, though, if a fix for this particular problem did not surface by the time the 3.14 merge window opens. Locking problems are hard enough to deal with when the locking primitives have simple and easily understood behavior; having subtle traps built into that layer of the kernel is a recipe for a lot of long-term pain.
