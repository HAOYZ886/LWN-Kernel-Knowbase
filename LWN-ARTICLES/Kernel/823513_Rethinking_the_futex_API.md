---
title: Rethinking the futex API
url: https://lwn.net/Articles/823513/
date: "June 18, 2020"
category: Futex
author: "By Jonathan Corbet June 18, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 18, 2020

The Linux [`futex()`](<https://www.man7.org/linux/man-pages/man2/futex.2.html>) system call is a bit of a strange beast. It is widely used to provide low-level synchronization support in user space, but there is no wrapper for it in the GNU C Library. Its implementation was meant to be simple, but kernel developers have despaired at the complex beast that it has become, and few dare to venture into that code. Recently, though, a new effort has begun to rework futexes; it is limited to a new system-call interface for now, but the plans go far beyond that. 

There is a wide range of synchronization options within the kernel, but there have always been fewer options in user space. For years, the only real option was [System V semaphores](<https://www.man7.org/linux/man-pages/man7/sysvipc.7.html>), but it is fair to say that they have never been universally loved. Developers have long wanted a mutex-like option for user space that does not kill performance. 

Back in 2002, Rusty Russell proposed a [fast user-space mutex mechanism](<https://lore.kernel.org/lkml/E16gRe3-0006ak-00@wagner.rustcorp.com.au/>) that quickly became known as a "futex"; this feature was present in the 2.6.0 kernel release at the end of 2003 and immediately used to control concurrency for POSIX threads. The initial implementation was just a few hundred lines of code. At its core, a futex is a 32-bit word of memory shared between cooperating processes; a value of one indicates that the futex is available, while anything else marks it as unavailable. A process wishing to acquire a futex will issue a locked decrement instruction, then verify that the resulting value was zero; if so, the acquisition was successful and execution can continue. Releasing the futex is a simple matter of incrementing its value again. 

The nice thing about futexes as described so far is that the kernel is not involved in their operation at all; futexes can be acquired and released without the need to make system calls. That cannot be sustained if there is contention for the futex, though; at that point, a task will have to block to wait for the futex to become available. That is where the `futex()` system call comes in: 

```
int futex(int *uaddr, int futex_op, int val,
                  const struct timespec *timeout,   /* or: uint32_t val2 */
                  int *uaddr2, int val3);
```

The initial `futex()` implementation had only two arguments: `uaddr` (the address of the futex) and `futex_op`, which would be either `+1` to increment the futex, or `-1` to decrement it. The modern equivalents for `futex_op` are `FUTEX_WAKE` (to signal that the futex has been freed and wake task(s) waiting for it) or `FUTEX_WAIT` to block until the futex becomes available. 

Many other operations also exist at this point. Over time, the futex interface has gained complexity, including "[robust futexes](<https://lwn.net/Articles/172149/>)", [adaptive spinning](<https://lwn.net/Articles/387246/>), [priority inheritance](<https://lwn.net/Articles/178253/>), and much more. See [this article](<https://lwn.net/Articles/360699/>) for a (somewhat dated) overview, the above-linked man page, or [this low-level description](<https://www.man7.org/linux/man-pages/man7/futex.7.html>) for more information. 

The current effort to rework futexes appears to be driven by a couple of concerns. One that goes mostly unstated is the desire to create a system-call interface that makes a bit more sense than `futex()`, which is a complex, multiplexed API with wildly varying arguments and a number of special cases. Whenever a system call is documented in terms like this: 

For several blocking operations, the timeout argument is a pointer to a timespec structure that specifies a timeout for the operation. However, notwithstanding the prototype shown above, for some operations, the least significant four bytes of this argument are instead used as an integer whose meaning is determined by the operation. 

One can conclude with a fair degree of certainty that the API design is not as great as it could be. 

In past years, when C-library developers have [discussed](<https://lwn.net/Articles/655044/>) exposing `futex()`, they have proposed splitting it into a set of simpler wrapper functions; that work has never been merged. Now, though, the [`futex2()` proposal](<https://lwn.net/ml/linux-kernel/20200612185122.327860-1-andrealmeid@collabora.com/>) from André Almeida does the same thing, adding three new system calls: 

```
int futex_wait(void *uaddr, unsigned long val, unsigned long flags, ktime_t *timeout);
        int futex_wait_time32(void *uaddr, unsigned long val, unsigned long flags, 
    			  old_time32_t *timeout);
        int futex_wake(void *uaddr, unsigned long nr_wake, unsigned long flags);
```

It is a rare patch set that includes a question like: ""Has anyone started worked on a implementation of this interface?"". Almeida's patch set adds no new functionality; indeed, it is currently rather less functional than the existing `futex()` API, lacking support for features like priority inheritance. Basic futex functionality is implemented, though, by calling into the existing `futex()` code. 

The purpose of this patch set is clearly not to show off new features at this point; instead, the hope is to nail down what a new futex API should look like, with the new features to be added after that is done. That said, there are some enhancements that the developers have in mind and would like to get into place. 

One of those is the ability to wait on multiple futexes at once and be awakened when any one of them becomes available. Gabriel Krisman Bertazi [posted an implementation of this functionality](<https://lwn.net/ml/linux-kernel/20190730220602.28781-2-krisman%40collabora.com/>) (for `futex()`) in July 2019; it is driven in particular by the needs of [Wine](<https://www.winehq.org/>), which is emulating a similar Windows feature. In a discussion sparked by another posting of this patch set in March, Thomas Gleixner [gently suggested](<https://lwn.net/ml/linux-kernel/87tv3aflqm.fsf%40nanos.tec.linutronix.de/>) that perhaps the time had come to design a new futex interface where features like this could be added (and used) more easily. The current proposal is a direct result of that suggestion. 

That said, the proposed API doesn't handle multiple futexes, but the cover letter from the current series describes a planned addition: 

```
struct futex_wait {
    	void *uaddr;
    	unsigned long val;
    	unsigned long flags;
        };
    
        int futex_waitv(struct futex_wait *waiters, unsigned int nr_waiters,
    		    unsigned long flags, ktime_t *timeout);
```

Another upcoming feature is the ability to handle futexes in a number of common sizes, not just the 32-bit ones supported today. 

Then, there is the issue of performance on [NUMA](<https://en.wikipedia.org/wiki/Non-uniform_memory_access>) systems. The kernel must maintain internal data structures describing the futexes that are currently being waited on; if those structures are kept on the wrong NUMA node, futex operations can sustain a lot of remote-node cache misses, which slows them down considerably. See [this article](<https://lwn.net/Articles/685769/>) for details. Futexes are often used by threaded processes that are all running on the same NUMA node; their performance would be improved if the kernel kept its data structures on the same node. Thus, there is a "NUMA hint" functionality planned for the new API that would suggest that the kernel keep its associated data structures on a specific node. 

While the thinking about how to improve futex functionality in the kernel has clearly entered a new phase, users should not hold their collective breath waiting for new futex features to enter production kernels. The complexity of this subsystem makes developers reluctant to push through quick changes; they have learned the hard way that it's easy for things to go wrong with futexes. So the new API and the implementation of any new features are likely to go through an extended period of review and revision. The "F" in "futex" may stand for "fast", but the implementation of new futex features may not be.
