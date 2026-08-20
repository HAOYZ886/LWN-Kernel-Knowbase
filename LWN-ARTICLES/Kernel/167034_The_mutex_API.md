---
title: The mutex API
url: https://lwn.net/Articles/167034/
date: "January 10, 2006"
category: "Locking mechanisms-Mutexes; Semaphores"
author: 
---

The mutex code may well have set a record for the shortest time spent in -mm for such a fundamental patch. It would not have been surprising for mutexes to sit in -mm through at least one kernel cycle, which would have had them being merged in or after 2.6.17. But the mutex code appeared in exactly one -mm release ([2.6.15-mm2](<http://lwn.net/Articles/166762/>), released on January 7) before being merged into the mainline on January 9\. 

The actual mutex type (minus debugging fields) is quite simple: 

```
struct mutex {
    	atomic_t		count;
    	spinlock_t		wait_lock;
    	struct list_head	wait_list;
        };
```

Unlike semaphores, mutexes have one definition which is used on all architectures. Some of the actual locking and unlocking code can be overridden if it can be made to perform better on a specific architecture, but the core data structure remains the same. The `count` field contains the state of the mutex. A value of one indicates that it is available, zero means locked, and a negative value means that it is locked and processes might be waiting. Separating the two "locked" cases is worthwhile: in the (usual) case where nobody is waiting for the mutex, there is no need to go through the process of seeing if anybody needs to be waked up. `wait_lock` controls access to `wait_list`, which is a simple list of processes waiting on the mutex. 

The mutex API (obtained through `<linux/mutex.h>`) is simple. Every mutex must first be initialized either at declaration time with: 

```
DEFINE_MUTEX(name);
```

Or at run time with: 

```
mutex_init(struct mutex *lock);
```

Once a mutex has been initialized, it can be locked with any of: 

```
void mutex_lock(struct mutex *lock);
        int mutex_lock_interruptible(struct mutex *lock);
        int mutex_trylock(struct mutex *lock);
```

A call to `mutex_lock()` will lock the mutex, putting the calling process into an uninterruptible wait if need be. `mutex_lock_interruptible()` uses an interruptible sleep; if the lock is obtained, it will return zero. A return value of `-EINTR` means that the locking attempt was interrupted by a signal and the caller should act accordingly. Finally, `mutex_trylock()` will attempt to obtain the lock, but will not sleep; unlike `mutex_lock_interruptible()`, it returns zero on failure (the lock was unavailable) and one if the lock is acquired. 

In all cases, the mutex must eventually be freed (by the same process which acquired it) through a call to: 

```
void mutex_unlock(struct mutex *lock);
```

Note that `mutex_unlock()` cannot be called from interrupt context. This restriction appears to have more to do with keeping mutexes from ever being used as completions than a fundamental restriction caused by the mutex design itself. Note also that a mutex can only be locked once - locking calls do not nest. 

Finally, there is a function for querying the state of a mutex: 

```
int mutex_is_locked(struct mutex *lock);
```

This function will return a boolean value indicating whether the mutex is locked or not, but will not change the state of the lock. 

Now that this code has been merged, the `semaphore` type can officially be considered to be on its way out. New code should not use semaphores, and old code which uses semaphores as mutexes should be converted over when an opportunity presents itself. The reader/writer semaphore type (rwsem) is a different beast, and is not affected by this patch. There is a debugging option which can be configured into development kernels which may help with the transition; with this option enabled, quite a few types of errors will be detected. 

At this point, code which uses the counting feature of semaphores lacks a migration path. There is evidently a plan to introduce a new, architecture-independent type for these users, but that code has not yet put in an appearance. Once that step has been taken, the path will be clear for the eventual removal of semaphores from the kernel entirely.
