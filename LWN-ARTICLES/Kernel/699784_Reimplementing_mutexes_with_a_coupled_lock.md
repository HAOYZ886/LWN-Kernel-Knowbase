---
title: Reimplementing mutexes with a coupled lock
url: https://lwn.net/Articles/699784/
date: "September 8, 2016"
category: "Locking mechanisms-Mutexes"
author: "By Jonathan Corbet September 8, 2016"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 8, 2016

Oscar Wilde once famously observed that fashion ""is usually a form of ugliness so intolerable that we have to alter it every six months"". Perhaps the same holds true of locking primitives in the kernel; basic mechanisms like the mutex have been through many incarnations over the years. This season, it would appear that [coupled atomic locks](<https://lwn.net/Articles/699094/>) are all the rage with the trendiest kernel developers, so it should not be surprising that a new mutex implementation using those locks is making the rounds. This code may be glittering and shiny, but it also has the potential to greatly simplify the mutex implementation. 

A mutex is a sleeping lock, meaning that kernel code that tries to acquire a contended mutex may go to sleep to wait until that mutex becomes available. Early mutex implementations would always put a waiter to sleep, but, following the scalability trends of the day, mutexes soon gained a glamorous accessory: optimistic spinning. Waking a sleeping thread can take a long time and, once that thread gets going, it may find that the processor cache contains none of its data, leading to unfashionable cache misses. A thread that spins waiting for a mutex, instead, will be able to grab it quickly once it becomes available and will likely still be cache-hot. Enabling optimistic spinning can improve performance considerably. There is a cost, in that mutexes are no longer fair (they can be "stolen" from a thread that has been waiting for longer), but being properly _à la mode_ is never free. 

Optimistic spinning brings with it an interesting complication, though, in that it requires tracking the current owner of the mutex. If that owner sleeps, or if the owner changes while a thread is spinning, it doesn't make any sense to continue spinning, since the wait is likely to be long. As a field within the mutex, the owner information is best protected by the mutex itself. But, by its nature, this information must be accessed by threads that do not own the mutex. The result is some tricky code that is trying to juggle the lock itself and the owner information at the same time. 

Peter Zijlstra has sent [an alternative mechanism](<https://lwn.net/Articles/699566/>) down the runway; it takes care of this problem by combining the owner information and lock status into a single field within the mutex. In current kernels, the `count` field, an `atomic_t` value, holds the status of the lock itself, while `owner`, a pointer to `struct task_struct`, indicates which thread owns the mutex. Peter's patch removes both of those fields, replacing them with a single `atomic_long_t` value called "`owner`". 

This value is 64 bits wide, large enough to hold a pointer value. If the mutex is available, there is no owner, so the new `owner` field contains zero. When the mutex is taken, the acquiring thread's `task_struct` pointer is placed there, simultaneously indicating that the mutex is unavailable and which thread owns it. The `task_struct` structure must be properly aligned, though, meaning that the bottom bits of a pointer to it will always be zero, so those bits are available for other locking-related purposes. Following this season's coupled-lock trend, two of those bits are so used, in ways that will be described shortly. 

With the new organization, the code to attempt to acquire a mutex now looks like this: 

```
static inline bool __mutex_trylock(struct mutex *lock)
        {
        	unsigned long owner, curr = (unsigned long)current;
        
        	owner = atomic_long_read(&lock->owner);
        	for (;;) { /* must loop, can race against a flag */
        	    unsigned long old;
        
        	    if (__owner_task(owner))
        		return false;
        	    old = atomic_long_cmpxchg_acquire(&lock->owner, owner,
        					      curr | __owner_flags(owner));
        	    if (old == owner)
        		return true;
        	    owner = old;
        	}
        }
```

The `__owner_task()` and `__owner_flags()` macros simply mask out the appropriate parts of the `owner` field. The key is the `atomic_long_cmpxchg_acquire()` call, which attempts to store the current thread as the owner of the mutex on the assumption that it is available. Should some other thread own the mutex, that call will fail, and the mutex code will know that it will have to work harder. 

There are currently two flags that can be stored in the least significant bits of `owner`. If a thread finds it must sleep while waiting for a contended mutex, it will set `MUTEX_FLAG_WAITERS`; the thread currently holding the mutex will then know it must wake the waiters when the mutex is freed. Most of the time, it is hoped, there will be no waiters; maintaining this bit allows for a bit of unnecessary work to be skipped. 

As mentioned above, optimistic spinning, while good for performance, is unfair; in the worst case, an unlucky thread contending for a highly contended mutex could be starved for a long time. In an attempt to prevent that problem, the second `owner` bit, `MUTEX_FLAG_HANDOFF`, can be used to change how a contended mutex changes ownership. 

If a thread tries and fails to obtain a mutex after having already slept waiting for it to become available, it can set `MUTEX_FLAG_HANDOFF` prior to returning to sleep. Later on, when the mutex is freed, the freeing thread will notice the flag and behave differently. In particular, it must avoid clearing the `owner` field as it normally would, lest some other thread, spinning on the mutex, steal it away. Instead, it finds the first thread in the wait queue for the mutex and transfers ownership directly, waking that thread once the job is done. This dance restores some fairness, at the cost of making everybody wait for the sleeping thread to wake up and get its work done. 

The new code simplifies the mutex implementation considerably by getting rid of a number of strange cases involving the separate `count` and `owner` fields. But it gets a bit better than that, since the new code is also architecture-independent; all of the old, architecture-specific mutex code can go away. So the bottom line of Peter's cover letter reads: 

```
49 files changed, 382 insertions(+), 1407 deletions(-)
```

Removing code, as it happens, is always in fashion, and removing 1000 lines of tricky assembly-language locking code is especially chic. Assuming that this code manages to avoid introducing performance regressions, it could be a must-have item at a near-future merge-window ball.
