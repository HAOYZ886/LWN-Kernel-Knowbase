---
title: Introducing lockrefs
url: https://lwn.net/Articles/565734/
date: "September 4, 2013"
category: Lockref; Reference counting; cmpxchg
author: "By Jonathan Corbet September 4, 2013"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 4, 2013

Reference counts are often used to track the lifecycle of data structures within the kernel. This counting is efficient, but it can lead to a lot of cache-line bouncing for frequently-accessed objects. The cost of this bouncing is made even worse if the reference count must be protected by a spinlock. The 3.12 kernel will include a new locking primitive called a "lockref" that, by combining the spinlock and the reference count into a single eight-byte quantity, is able to reduce that cost considerably. 

In many cases, reference counts are implemented with `atomic_t` variables that can be manipulated without taking any locks. But the lockless nature of an `atomic_t` is only useful if the reference count can be changed independently of any other part of the reference-counted data structure. Otherwise, the structure as a whole must be locked first. Consider, for example, the heavily-used `dentry` structure, where reference count changes cannot be made if some other part of the kernel is working with the structure. For this reason, `struct dentry` prior to 3.12 contains these fields: 

```
unsigned int d_count;	/* protected by d_lock */
        spinlock_t d_lock;		/* per dentry lock */
```

Changing `d_count` requires acquiring `d_lock` first. On a system with a filesystem-intensive workload, contention on `d_lock` is a serious performance bottleneck; acquiring the lock for reference count changes is a significant part of the problem. It would thus be nice to find a way to to avoid that locking overhead, but it is not possible to use atomic operations for `d_count`, since any thread holding `d_lock` must not see the value of `d_count` change. 

The "lockref" mechanism added at the beginning of the 3.12 merge window allows mostly-lockless manipulation of a reference count while still respecting an associated lock; it was originally [implemented](<https://lwn.net/Articles/562249/>) by Waiman Long, then modified somewhat by Linus prior to merging. A lockref works by packing the reference count and the spinlock into a single eight-byte structure that looks like: 

```
struct lockref {
    	union {
    	    aligned_u64 lock_count;
    	    struct {
    		spinlock_t lock;
    		unsigned int count;
    	    };
        	};
        };
```

Conceptually, the code works by checking to be sure that the lock is not held, then incrementing (or decrementing) the reference count while verifying that no other thread takes the lock while the change is happening. The key to this operation is the magic `cmpxchg()` macro: 

```
u64 cmpxchg(u64 *location, u64 old, u64 new);
```

This macro maps directly to a machine instruction that will store the `new` value into `*location`, but only if the current value in `*location` matches `old`. In the lockref case, the `location` is the `lock_count` field in the structure, which holds both the spinlock and the reference count. An increment operation will check the state of the lock, compute the new reference count, then use `cmpxchg()` to atomically store the new value, insuring that neither the count nor the lock has changed in the meantime. If things _do_ change, the code will either try again or fall back to old-fashioned locking, depending on whether the lock is free or not. 

This trickery allows reference count changes to be made (most of the time) without actually acquiring the spinlock and, thus, without contributing to lock contention. The associated performance improvement can be impressive — a [factor of six](<https://lwn.net/Articles/565737/>), for example, with one of Waiman's benchmarks testing filesystem performance on a large system. Given that the new lockref code is only being used in one place (the dentry cache), that is an impressive return from a relatively small amount of changed code. 

At the moment, only 64-bit x86 systems have a full lockref implementation. It seems likely, though, that other architectures will gain support by the end of the 3.12 development cycle, and that lockrefs will find uses in other parts of the kernel in later cycles. Meanwhile, the focus on lock overhead has led to [improvements](<https://lwn.net/Articles/565738/>) elsewhere in the filesystem layer that should make their way in during this merge window; it has also drawn attention to some other places where the locking can clearly be improved with a bit more work. So, in summary, we will see some significant performance improvements in 3.12, with more to come in the near future.
