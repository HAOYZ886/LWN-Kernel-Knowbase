---
title: Unified spinlock initialization
url: https://lwn.net/Articles/109505/
date: "November 3, 2004"
category: Spinlocks
author: 
---

There have traditionally been two ways to initialize a spinlock inside the kernel. It can be done with an explicit assignment: 

```
spinlock_t lock = SPIN_LOCK_UNLOCKED;
```

or with a function call: 

```
spinlock_t lock;
    	spin_lock_init(&lock);
```

Linus has recently merged a set of patches which move all in-kernel initializations over to the function-based form. There has been no patch to remove the `SPIN_LOCK_UNLOCKED` macro, but it is not hard to see a move in that direction once the conversion is complete. 

The stated reasons for this change include consistency and making life easier for automatic lock validators. There is also an unstated, but evident reason: the assignment form of lock initialization gets in the way of the realtime preemption patches. Those patches change most spinlocks in the kernel to a different, mutex type, and that breaks the initializers. As a result, the preemption patches must change all of those initializations throughout the kernel. By putting those specific changes into the mainline, it is possible to make the realtime patches smaller, less intrusive, and a little bit less scary.
