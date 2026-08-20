---
title: Generic semaphores
url: https://lwn.net/Articles/273731/
date: "March 17, 2008"
category: Semaphores
author: "By Jonathan Corbet March 17, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 17, 2008

Most kernel patches delete some code, replacing it with newer and (presumably) better code. Much of the time, it seems, the new code is more voluminous than what came before. Occasionally, though, a patch comes along which deletes over 7600 lines of code - replacing it with a mere 314 lines - while claiming to maintain the same functionality. Matthew Wilcox's [generic semaphore patch](<http://lwn.net/Articles/273657/>) is one of those changes. 

In essence, a semaphore is a counter with a wait queue attached to it. When kernel code wants to access the resource protected by the semaphore, it makes a call to: 

```
void down(struct semaphore *sem);
```

This call will check the counter associated with `sem`; if it is greater than zero, the counter will be decremented and control returns to the caller. Otherwise the caller will be put to sleep until sometime in the future when the counter has been increased again. Increasing the counter - when the the protected resource is no longer needed - is done with a call to `up()`. Semaphores can be used in any situation where there is a need to put an upper limit on the number of processes which can be within a given critical section at any time. In practice, that upper limit is almost always set to one, resulting in semaphores which are used as a straightforward mutual exclusion primitive. 

In current kernels, semaphores are implemented with highly-optimized, architecture-specific code. There are, in fact, more than twenty independent semaphore implementations in the kernel code base. Matthew's patch rips all of that out and replaces it with a single, generic implementation which works on all architectures. After the patch is applied, a semaphore looks like this: 

```
struct semaphore {
    	spinlock_t		lock;
    	int			count;
    	struct list_head	wait_list;
        };
```

The implementation follows from this definition in a straightforward way: the spinlock is used to protect manipulations of `count`, while `wait_list` is used to put processes to sleep when they must wait for `count` to increase. The actual code, of course, is somewhat complicated by performance and interrupt-safety considerations, but it remains relatively short and simple. 

One might ask: why weren't semaphores done this way in the first place? The answer is that, once upon a time (prior to 2.6.16), semaphores were one of the primary mutual exclusion mechanisms in the kernel. The 2.6.16 cycle brought in mutexes from the realtime tree, and most semaphore users were converted over. So semaphores, which were once a performance-critical primitive, are now much less so. As a result, any need there may have been for carefully hand-tuned, architecture-specific code is gone. So the code might as well go too. 

The other question which comes up is: why are semaphores still being used at all? The number of semaphore users has dropped considerably since 2.6.16, but there are still a number of them in the kernel. Some of those could certainly be converted to mutexes, but doing so requires a careful audit of the code to be sure that the semaphore's counting feature is not being used. Once that work is done, it may turn out that, in some places, a semaphore is truly the right data structure. So semaphores are likely to remain - but they'll require rather less code than before.
