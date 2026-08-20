---
title: Reworking the semaphore interface
url: https://lwn.net/Articles/163842/
date: "December 14, 2005"
category: "Locking mechanisms-Mutexes; Semaphores"
author: 
---

The Linux kernel contains a full counting semaphore implementation. Given a semaphore, a call to `down()` will sleep until the semaphore contains a positive value, decrement that value, and return. Calling `up()` increments the semaphore's value and wakes up a process waiting for the semaphore, if one exists. If the initial value of the semaphore is ten, then ten different threads can call `down()` without blocking. 

Most users of semaphores do not use the counting feature, however. Instead, they initialize the semaphore to a value of one, allowing a single thread to hold the semaphore at any given time. This mode of use turns a semaphore into a "mutex," a mutual exclusion primitive which can be used to implement critical sections. Using a semaphore in this way is entirely valid. 

There is one little issue, however: a simple binary mutex can often be implemented more cheaply than a full counting semaphore. If a semaphore is used in the mutex mode, the extra cost of the counting capability is simply wasted. Linux semaphores also suffer from highly architecture-dependent implementations, to the point that any changes to the semaphore API are very difficult to make. So cleaning up semaphores has been one of those items on the "do to" list for some time. 

David Howells [went ahead and did it](<http://lwn.net/Articles/163807/>). His patch adds a new, binary mutex type to the kernel. Since almost all of the semaphores currently in use are, in reality, mutexes, David changed the prototypes of most of the semaphore functions (`down()` and variants, `up()`, `init_MUTEX()`, `DECLARE_MUTEX()`) to take a mutex rather than a semaphore. To make things work again, most semaphore declarations have been changed to `struct mutex`, but, beyond the declaration change, code using mutexes need not be modified. 

For code which truly needs a semaphore, a new set of functions has been provided: 

```
void down_sem(struct semaphore *sem);
        void up_sem(struct semaphore *sem);
        int down_sem_trylock(struct semaphore *sem);
        ...
```

Kernel code which was actually using the counting capability of semaphores has been changed to use the new functions. 

This patch makes fundamental changes to the kernel's mutual exclusion mechanisms, creates a flag day which breaks all out-of-tree code, and is generally quite large. But there is surprisingly little resistance to the patch in general. Some developers are concerned that some counting semaphores may have been converted to mutexes erroneously - it is hard to audit that much code and be absolutely sure of how every semaphore is used. It has also been noted that the posted mutex implementation may actually be slower than the semaphores it replaces, but that is something which, it is assumed, can be fixed. In general, however, almost nobody objects to making this sort of change. 

There are some disagreements over just how the change should be done, however. Some developers do not want to see the old `down()` and `up()` functions switched to a different type which has no counter to bump "down" or "up." The alternative would be to create a completely new API for mutexes; Alan Cox has [suggested](<https://lwn.net/Articles/163867/>) names like `sleep_lock()` and `sleep_unlock()`. A completely new API would make it clear what is really going on; it would also make it possible to change over users gradually as they are audited. 

Some developers would rather see a big flag day than a year-long series of patches slowly converting semaphore users over to mutexes. For them, the mutex changeover is a chance to get the API right, and they would rather see everything changed over at once. Gradual changeovers, it is argued, never seem to come to a conclusion; examples include the continued existence of the big kernel lock and the long-deprecated `sleep_on()` functions. Rather than live with a deprecated API for years, it may be better to just take the pain all at once and be done with it. 

It has also been pointed out that there is another mutex patch in circulation: the real-time preemption tree has had mutexes for the last year. So far, there has been no real debate on whether the -rt implementation is better; Ingo Molnar does not seem to be pushing it, even though this might be a good opportunity to merge a significant chunk of the -rt tree into the mainline. 

In the end, it looks like some sort of mutex patch is likely to be merged into a future mainline kernel - though it almost certainly will not be ready when the 2.6.16 window opens. The form of that patch could change significantly, however; stay tuned.
