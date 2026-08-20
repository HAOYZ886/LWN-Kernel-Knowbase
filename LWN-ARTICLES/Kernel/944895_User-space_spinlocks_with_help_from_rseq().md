---
title: "User-space spinlocks with help from rseq()"
url: https://lwn.net/Articles/944895/
date: "September 22, 2023"
category: "Spinlocks-User-space"
author: "By Jonathan Corbet September 22, 2023 OSSEU"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 22, 2023

* * *

[OSSEU](<https://lwn.net/Archives/ConferenceByYear/#2023-Open_Source_Summit_Europe>)

Back in May, André Almeida [presented some work](<https://lwn.net/Articles/931789/>) toward the creation of user-space spinlocks using adaptive spinning. At that time, the work was stalled because there is, in Linux, currently no way to quickly determine whether a given thread is actually executing on a CPU. Some progress has since been made on that front; at the [2023 Open Source Summit Europe](<https://events.linuxfoundation.org/open-source-summit-europe/>), Almeida returned to discuss how that difficulty might be overcome. 

[![\[André Almeida\]](https://static.lwn.net/images/conf/2023/osseu/AndreAlmeida-sm.png)](<https://lwn.net/Articles/944896/>) He started with on overview of locking primitives and how spinlocks, in particular, work. In short, a spinlock is so-named because, if it an attempt to acquire a lock fails, the code will recheck its status in a loop (thus "spinning") until the lock becomes available. Spinlocks are relatively easy to implement in the kernel because, by the rules under which spinlocks operate, the holder of a lock is known to be running on a CPU somewhere in the system and should release it soon; that insures that the CPU time lost to spinning will be small. 

In user space, the story is more complex. One thread might be spinning on a lock while the holder has been preempted and isn't running at all. In such cases, the lock will not be released soon, and the spinning just wastes CPU time. In the worst case, the thread that is spinning may be the one that is keeping the lock holder from running, meaning that the spinning thread is actively preventing the lock it needs from being released. In such situations, the code should simply stop spinning and go to sleep until the lock is released. 

Doing that, though, requires a way for the lock-acquisition code to know that the lock owner is not running. One could add a system call for that purpose, but system calls are expensive; in this case, the system-call overhead might easily overwhelm the time spent in the critical section protected by the lock. If it is necessary to call into the kernel, it is better to just block until the lock is released. What is really needed is a way to gain that information _without_ making a system call. 

In the May discussion, the idea of using the [restartable sequences](<https://lwn.net/Articles/697979/>) feature to gain that information came up. This subsystem has hooks into the scheduler to track events like task preemption; it also uses a shared-memory segment to communicate some of that information to user space. Perhaps restartable sequences could be employed to solve this problem as well? 

The maintainer of the restartable sequences code, Mathieu Desnoyers, [quickly responded](<https://lwn.net/ml/linux-kernel/20230529191416.53955-1-mathieu.desnoyers@efficios.com/>) with a patch to implement this functionality. This patch adds a new structure member to the `rseq` struct that is shared between the kernel and user space: 

```
struct rseq_sched_state {
    	/*
    	 * Version of this structure. Populated by the kernel, read by
    	 * user-space.
    	 */
    	__u32 version;
    	/*
    	 * The state is updated by the kernel. Read by user-space with
    	 * single-copy atomicity semantics. This field can be read by any
    	 * userspace thread. Aligned on 32-bit. Contains a bitmask of enum
    	 * rseq_sched_state_flags. This field is provided as a hint by the
    	 * scheduler, and requires that the page holding this state is
    	 * faulted-in for the state update to be performed by the scheduler.
    	 */
    	__u32 state;
    	/*
    	 * Thread ID associated with the thread registering this structure.
    	 * Initialized by user-space before registration.
    	 */
    	__u32 tid;
        };
```

The `state` field, which holds a set of flags describing the execution state of the process in question, is the key here. There is only one flag, `RSEQ_SCHED_STATE_FLAG_ON_CPU`, defined. Whenever the thread associated with this structure is placed onto a CPU for execution, this flag will be set; if the thread stops running for any reason, the flag is cleared again. 

This information is enough for the implementation of adaptive spinning in user space. If an attempt to acquire a spinlock fails, the first step is to check the `rseq_sched_state` of the thread holding the lock (this implicitly requires that this communication is happening between threads that can access each other's restartable-sequences state). If that check shows that the thread is running, then it makes sense to spin waiting for the lock to be freed (with a check inside the loop, of course, to detect the case where the holder is subsequently preempted). Otherwise, a system call is made to simply block until the lock is freed. 

That said, Almeida concluded by saying that he is still not entirely sure if this idea lives up to its potential. There is work to be done to optimize cache behavior, integrate adaptive spinning into the POSIX threads locking primitives, and do a lot of benchmarking work. But the approach appears to have promise, and the rest is just work. 

[Thanks to the Linux Foundation for supporting our travel to this event.]
