---
title: Wait/wound mutexes
url: https://lwn.net/Articles/548909/
date: "May 1, 2013"
category: "Locking mechanisms-Mutexes"
author: "By Jonathan Corbet May 1, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 1, 2013

Developers wanting to add new locking primitives to the kernel tend to be received with a certain amount of skepticism. The kernel is already well equipped with locking mechanisms, and experience shows that new mechanisms tend to be both unnecessary and hard to get right. The "[wait/wound mutex mechanism](<https://lwn.net/Articles/548780/>)" proposed by Maarten Lankhorst may well get that kind of response. But it is an interesting approach to a specific locking problem that merits a closer look. 

#### A conceptual overview

Situations where multiple locks must be held simultaneously pose a risk of deadlocks: if the order in which those locks are acquired is not always the same, there will eventually come a time when two threads find themselves blocked, each waiting for the other to release a lock. Kernel code tends to be careful about lock ordering, and the "lockdep" checking tool has gotten quite good about finding code that violates the rules. So deadlocks are quite rare, despite the huge number of locks used by the kernel. 

But what about situations where the ordering of lock acquisition cannot be specified in advance, or, even worse, is controlled by user space? Maarten's patch describes one such scenario: a chain of buffers used with the system's graphical processing unit (GPU). These buffers must, at various times, be "owned" by the GPU itself, the GPU driver, user space, and, possibly, another driver completely, such as for a video frame grabber. User space can submit the buffers for processing in an arbitrary order, and the GPU may complete them in a different order. If locking is used to control the ownership of the buffers, and if multiple buffers must be manipulated at once, avoiding deadlocks could become difficult. 

Imagine a simple situation where there are two buffers of interest: 

> ![\[buffers\]](https://static.lwn.net/images/2013/wake-wound1.png)

Imagine further that we have two threads (we'll call them T1 and T2) that attempt to lock both buffers in the opposite order: T1 starts with Buffer A, while T2 starts with Buffer B. As long as they do not both try to grab the buffers at the same time, things will work. But, someday, each will succeed in locking one buffer and a situation like this will develop: 

> ![\[locked buffers\]](https://static.lwn.net/images/2013/wake-wound2.png)

The kernel's existing locking primitives have no answer to a situation like this other than "don't do that." The wait/wound mutex, instead, is designed for just this case. In general terms, what will happen in this situation is: 

  * The thread that "got there first" will simply sleep until the remaining buffer becomes available. If T1 started the process of locking the buffers first, it will be the thread that waits. 

  * The other thread will be "wounded," meaning that it will be told it must release any locks it holds and start over from scratch. 

So if T2 is wounded, the deadlock will be resolved by telling T2 to release Buffer B; it must then wait until that buffer becomes available again and start over. So the situation will look something like this: 

> ![\[locked buffers\]](https://static.lwn.net/images/2013/wake-wound3.png)

Once T1 has released the buffers, T2 will be able to retry and, presumably, make forward progress on its task. 

#### The details

The first step toward using a set of locks within the wait/wound mechanism is to define a "class"; this class is essentially a context within which the locks are to be acquired. When multiple threads contend for the same locks, they must do so using the same class. A wait/wound class is defined with: 

```
#include <linux/mutex.h>
    
        static DEFINE_WW_CLASS(my_class);
```

As far as users of the system are concerned, the class needs to exist, but it is otherwise opaque; there is no explicit initialization required. Internally, the main purpose for the class's existence is to hold a sequence number (an atomic counter) used to answer the "who got there first" question; it also contains some information used by lockdep to verify correct use of wait/wound locks. 

The acquisition of a specific set of locks must be done within a "context" that tracks the specific locks held. Before acquiring the first lock, a call should be made to: 

```
void ww_acquire_init(struct ww_acquire_ctx *ctx, struct ww_class *ww_class);
```

This call will assign a sequence number to the context and do a bit of record keeping. Once that has been done, it is possible to start acquiring locks: 

```
int ww_mutex_lock(struct ww_mutex *lock, struct ww_acquire_ctx *ctx);
```

If the lock has been successfully acquired, the return value will be zero. When all goes well, the thread will manage to acquire all of the locks it needs. Once that process is complete, that fact should be signaled with: 

```
void ww_acquire_done(struct ww_acquire_ctx *ctx);
```

This function is actually a no-op in the current implementation, but that could change in the future. After this call, the processing of the locked data can proceed normally. Once the job is done, it is time to release the locks and clean up: 

```
void ww_mutex_unlock(struct ww_mutex *lock);
        void ww_acquire_fini(struct ww_acquire_ctx *ctx);
```

Each held lock should be released with `ww_mutex_unlock()`; once _all_ locks have been released, the context should be cleaned up with `ww_acquire_fini()`. 

The above description describes what happens when all goes well, but it has left out an important case that all wait/wound mutex users must handle: the detection of a potential deadlock. That case comes about whenever an attempt is made to lock a `ww_mutex` that is already locked; in this case, there are three possible outcomes. 

The first of these comes about if the locking thread already holds that `ww_mutex` and is attempting to lock it for a second time. With ordinary mutexes, this would be an error, but the wait/wound mechanism is designed for this case. Evidently, sometimes, the ordering of the locking is _so_ poorly defined that multiple locking attempts can happen. In such cases, `ww_mutex_lock()` will return `-EALREADY`. The locking thread, assuming it knows how to respond to `-EALREADY`, can continue about its business. 

The second possibility is that the sequence number in the context for the locking process is higher than the number associated with thread already holding the lock. In this case, the new caller gets "wounded"; `ww_mutex_lock()` will return `-EDEADLK` to signal that fact. The wounded thread is expected to clean up and get out of the way. "Cleaning up" means releasing all locks held under the relevant context with calls to `ww_mutex_unlock()`. Once all of the locks are free, the wounded thread can try again, but only when the contended lock is released by the victorious thread; waiting for that to happen is done with: 

```
void ww_mutex_lock_slow(struct ww_mutex *lock, struct ww_acquire_ctx *ctx);
```

This function will block the calling thread until `lock` becomes free; once it returns, the thread can try again to acquire all of the other locks it needs. It is entirely possible that this thread could, once again, fail to acquire all of the needed locks. But, since the sequence number increases monotonically, a once-wounded thread must eventually reach a point where it has the highest priority and will win out. 

The final case comes about when the new thread's sequence number is lower than that of the thread currently holding the lock. In this case, the new thread will simply block in `ww_mutex_lock()` until the lock is freed. If the thread holding the contended lock attempts to acquire another lock that is already held by the new thread, it will get the `-EDEADLK` status at that point; it will then release the contended lock and let the new thread proceed. Going back to the example above: 

> ![\[locked buffers\]](https://static.lwn.net/images/2013/wake-wound2.png)

Thread T1, holding the lower sequence number, will wait for Buffer B to be unlocked, while thread T2 will see `-EDEADLK` when it attempts to lock Buffer A. 

The documentation in the patch does not describe what happens if the holding process never calls `ww_mutex_lock()` again. In this case, it will never know that it is supposed to back off. But, in this case, the holder must necessarily already have acquired all of the locks it needs, so there should be no reason why it cannot simply finish its work and release the locks normally. So the end result will be the same. 

#### Conclusion

Needless to say, there are plenty of details that have not been covered here; see the [ww-mutex-design.txt document](<https://lwn.net/Articles/548921/>) included with the patch set for more information. 

In that document, there are code examples for three different ways of working with wait/wound mutexes. One need not read for long to conclude that the API looks a bit complex and tricky to use; it will be far harder to write correct locking code using this facility than it would be with normal mutexes. Perhaps that complexity is necessary, and it seems certain that this mechanism will not be needed in many places in the kernel, so the complexity should not spread too far. But an API like this can be expected to raise some eyebrows. 

What is missing at this point is any real code that uses wait/wound mutexes. Kernel developers will certainly want to see some examples of where this kind of locking mechanism is needed. After all, the kernel has made it through its first two decades without this kind of complex locking; convincing the community that this feature is now necessary is going to take a strong sales effort. That is best done by showing how wait/wound mutexes solve a problem that cannot be easily addressed otherwise. Until that is done, wait/wound mutexes are likely to remain an interesting bit of code on the sidelines.
