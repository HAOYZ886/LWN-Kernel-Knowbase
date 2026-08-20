---
title: "Time for a new semaphore type?"
url: https://lwn.net/Articles/130506/
date: "April 5, 2005"
category: Semaphores
author: 
---

The Linux kernel uses two basic mutual exclusion primitives internally: spinlocks (which are fast, but require that critical sections be atomic) and semaphores (which are slower, but can sleep). These mechanisms are adequate for most uses, but there are exceptions. Trond Myklebust has encountered one of those exceptions when working on the NFSv4 code. In NFSv4, there are situations where non-atomic code must obtain a lock, but the thread cannot block at that point without risking deadlocks. So Trond set out to add an asynchronous capability to the Linux semaphore implementation - a way to request that a function be called at some point in the future when the semaphore becomes available. He encountered a little problem, however: each architecture implements its own, highly-optimized semaphore code, often in assembly language. To add functionality to semaphores, he would have to dig into more than 20 different implementations, and, somehow, ensure that they all still work afterward. 

Rather than dive into that jungle, Trond elected to start over. The result is [a new semaphore type](<https://lwn.net/Articles/130509/>) which Trond calls an "iosem." At its core, an iosem looks much like a regular semaphore: 

```
#include <linux/iosem.h>
    
        void iosem_init(struct iosem *sem);
        void iosem_lock(struct iosem *sem);
        void iosem_unlock(struct iosem *sem);
```

A call to `iosem_lock()` is similar to a call to `down()`; it will block until the semaphore is available. 

The definition of an `iosem` structure is simple: 

```
struct iosem {
    	unsigned long state;
    	wait_queue_head_t wait;
        };
```

Whenever a thread releases the lock, it will perform a wakeup on the given wait queue entry. For the synchronous locking case, that will cause the threads waiting for the lock to be scheduled; one of them will then succeed in acquiring that lock. Everything works as one might expect. 

2.6 wait queues are flexible things, however. In particular, it is possible to replace the function that is called when a wakeup occurs; this capability turns a wait queue into a fairly general notification mechanism. The iosem code takes advantage of this mechanism to allow different things to happen when an iosem becomes available. For example, consider this interface: 

```
struct iosem_work {
    	struct work_struct work;
    	struct iosem_wait waiter;
        };
    
        void iosem_work_init(struct iosem_work *work,
                             void (*func) (void *), void *data);
    
        int iosem_lock_and_schedule_work(struct iosem *sem,
                                         struct iosem_work *work);
```

A thread using this interface sets up a function (`func`), then calls `iosem_lock_and_schedule_work()`. If the iosem is available, `func` will be called immediately, with the lock held. Otherwise, a special entry will be added to the iosem's wait queue, and the call to `iosem_lock_and_schedule_work()` will return immediately. At some future time, `func` will be called (with the lock held) by way of a workqueue. Either way, `func` must release the lock when it is done. 

Other sorts of behavior could easily be added to this interface. Since the same code is used for all architectures, the iosem mechanism is relatively easy to extend. There has been some interest from maintainers of other parts of the kernel ([asynchronous I/O](<https://lwn.net/Articles/130514/>), for example) in using this mechanism. There have been a few complaints, however, about the name and about adding a wholly new mutual exclusion primitive to the kernel. In particular, Benjamin LaHaise (who has recently resurfaced on the kernel lists) has [stated](<https://lwn.net/Articles/130515/>) that it would be better to rationalize the current semaphore implementation \- and said that he would do the work. So, while an asynchronous semaphore implementation is likely to get into the kernel, the form it will take is not yet clear.
