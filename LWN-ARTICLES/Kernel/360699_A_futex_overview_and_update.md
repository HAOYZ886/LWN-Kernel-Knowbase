---
title: A futex overview and update
url: https://lwn.net/Articles/360699/
date: "November 11, 2009"
category: Futex
author: "November 11, 2009 This article was contributed by Darren Hart"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

November 11, 2009

This article was contributed by Darren Hart

The [futex [PDF]](<http://www.kernel.org/doc/ols/2002/ols2002-pages-479-495.pdf>) mechanism, introduced in 2.5.7 by Rusty Russell, Hubertus Franke, and Mathew Kirkwood, is a fast, lightweight kernel-assisted locking primitive for user-space applications. It provides for very fast uncontended lock acquisition and release. The futex state is stored in a user-space variable (an unsigned 32-bit integer on all platforms). Atomic operations are used in order to change the state of the futex in the uncontended case without the overhead of a syscall. In the contended cases, the kernel is invoked to put tasks to sleep and wake them up. 

Futexes are the basis of several mutual exclusion constructs commonly used in threaded programming. These include pthread mutexes, condvars, semaphores, rwlocks, and barriers. They have been through a lot of reconstructive and cosmetic surgery over the last several years, and are now more efficient, more functional, and better documented than ever before. 

#### Overview

While few application developers will use futexes directly, a cursory knowledge of how to do so is necessary to appreciate the improvements presented a bit later. By way of a simple example, futexes can be used to store the state of a lock and provide a kernel waitqueue for tasks blocking on the lock. To minimize syscall overhead, this state should allow for atomic lock acquisition when the lock is uncontended. The state could be defined as: 

  0. unlocked 
  1. locked 

In order to acquire the lock, an atomic test-and-set instruction (such as `cmpxchg()`) can be used to test for 0 and set to 1\. In this case, the locking thread acquires the lock without involving the kernel (and the kernel has no knowledge that this futex exists). When the next thread attempts to acquire the lock, the test for zero will fail and the kernel needs to be involved. The blocking thread can then use the `futex()` system call with the FUTEX_WAIT opcode to put itself to sleep on the futex, passing the address of the futex state variable as an argument. To release the lock, the owner changes the lock state to zero (unlocked) and issues the FUTEX_WAKE opcode, which will wake the blocked thread to return to user space and try to acquire the lock (as described above). This is a an obviously trivial example with lots of room for optimization. Ulrich Drepper's "[Futexes are Tricky](<http://people.redhat.com/drepper/futex.pdf>)" [PDF] is still the undisputed reference for using futexes to build locking primitives such as mutexes. It explores the many race conditions involved with using futexes as well as optimizations to improve on the example given here. 

When the user threads call into the kernel with the `futex()` system call, they pass the address of the futex state (uaddr), the opcode to perform (op), and various other arguments. The uaddr is used by the kernel to generate a unique "futex_key" to reference the futex. When a thread requests to block on a futex, as with FUTEX_WAIT, a "futex_q" is created and queued in the "futex_queues" hash table. There is one futex_q for every task blocked on a futex, possibly many futex_q's per futex. The futex_queues themselves (the hash table lists, not the "futex_q's") are shared among futexes, since multiple futex_keys will hash to the same queue. These relationships are illustrated below: 

![\[Futex diagram\]](https://static.lwn.net/images/ns/kernel/dvh-futexes.png)

In most cases, there is no policy defining how the user space state variable is to be used (despite what the futex man pages may or may not say). The application (or a library such as glibc) uses this value to define the state of the locking construct being implemented. This can be as simple as a boolean variable (as in the example above), however optimized implementations and other locking mechanisms require more complex state values. 

In addition to the simple FUTEX_WAIT and FUTEX_WAKE operations, the kernel also manages special operations that require more knowledge of the locking construct's state than can be had in user space, most notably the priority inheritance (PI) chains and robust lists. PI and robust futexes are exceptions to the user-defined-policy rule regarding the state variable. Their state depends not only on the locked state of the mutex, but also on the identity of the owner and whether or not there are waiters. As such, the futex value is defined as the thread identifier (TID) of the owner and a bit to indicate pending owners. This policy still allows for user space atomic operations to avoid calling into the kernel in the uncontended case. 

#### Improvements

Futexes have seen numerous improvements from a handful of developers since their debut appearance in 2.5.7. Some notable improvements include priority based wake-up for real-time tasks (by Pierre Peiffer) and [robust](<http://lwn.net/Articles/172149/>) and [PI futexes](<http://lwn.net/Articles/177111/>) (by Ingo Molnar and Thomas Gleixner). These latter features have been in use for some time and have been adequately covered here on LWN.net as well as in the excellent discussions on the kernel mailing list. Your author's foray into futexes picks up here, about two and a half years ago. Aside from several fixes to address rare corner cases and race conditions, the futex code has seen several functional and performance improvements since those earlier contributions. 

Significant effort went into reducing futex overhead. Eric Dumazet introduced [private futexes](<http://lwn.net/Articles/229668/>) as an optimization for PTHREAD_PROCESS_PRIVATE pthread mutexes. Private futexes can only be used by threads of the same process. They are distinguishable from each other simply by their virtual address, while shared futexes have different virtual addresses in each process, requiring the kernel to lookup their physical address for unique identification. This optimization eliminates the use of the `mmap_sem` semaphore for private futexes, reducing system-wide contention. It also eliminates the atomic operations used in reference counting for private futexes, resulting in less cache-line bouncing on SMP machines. Glibc now uses private futexes by default. 

Peter Ziljstra further reduced the futex dependency on `mmap_sem` by using `get_user_pages_fast()` in the fast paths, making use of `get_user_pages()`, and pushing the `mmap_sem` locks down tightly around the slow paths (September 2008). These changes had the added benefit of removing much of the virtual memory related logic from `kernel/futex.c`, simplifying the code considerably. Due to their dependence on user space addresses, futexes are burdened with several possible fault points. Holding `mmap_sem` complicated the fault logic since it had to be released prior to calling `get_user()`. With `mmap_sem` usage reduced, your author greatly simplified the fault logic (March 2009), resulting in far more legible code. 

Bitset conditional wakeup was added by Thomas Gleixner (February 2008) in order to enable optimized rwlock implementations in glibc. FUTEX_WAIT_BITSET and FUTEX_WAKE_BITSET allow the user to specify a bitmask which limits the woken tasks to those which specified the same bitset (or a superset, such as FUTEX_BITSET_MATCH_ANY) at wait time. 

Since the introduction of PI futexes, the glibc condvar implementation of `pthread_cond_broadcast()` (with a PI mutex) has been forced to wake all waiters, rather than take advantage of FUTEX_REQUEUE, due to the lack of support for requeueing to PI futexes. This leads to a wake-up storm as all the waiters race back to user space to contend for the lock. It also fails to ensure that the highest priority task acquires the lock first. Recent kernels (2.6.31-rt* and 2.6.32-rc*) now have your author's FUTEX_CMP_REQUEUE_PI patches (April 2009) which provide the kernel-side support for [requeueing waiters from a non-PI futex to a PI futex](<http://lwn.net/images/conf/rtlws11/papers/paper.10.html>). With glibc patches in the works by Dinakar Guniguntala, real-time applications will soon be able to use pthread condition variables with guaranteed wake-up order and fewer overall wake-ups. 

#### Now What?

While there are several items that futex developers may consider in the future, they are hopeful that `kernel/futex.c` and all its brain-bending, liver-killing insanity can be put to rest for at least a little while. However, since no article is complete without a list of next steps, the following items may receive some attention in the future: 

Man pages: The current man pages do not include some of the new futex operations. They suggest a policy for the value of the futex which has led to some confusion regarding usage of futexes. Worst of all, the user space `futex()` definition has been removed from `/usr/include/linux/futex.h`, rendering the man pages not only incomplete, but also inaccurate. Users of futexes must use the syscall interface directly. 

Adaptive futexes: It is possible that some of the scheduling overhead of futexes can be reduced by some optional amount of spinning prior to going to sleep in the kernel. However, as futexes expose their state to user space, this spinning can also be done in user space, as is done with adaptive mutexes in glibc now, albeit without the knowledge of whether the owner is running, so spinning is reduced to a simple maximum-retries loop. 

Interruptible futexes: There is some interest in interruptible blocking lock operations from large proprietary software projects. Futex operations currently restart themselves in the event of a signal, rather than returning -EINTR to user space. Futexes could be flagged with FUTEX_INTERRUPTIBLE which would be checked on signal-induced wakeup to determine if the syscall should be restarted or if -ECANCELED should be returned to user space. Exposing such a feature in the pthread locking primitives would involve non-POSIX compliant changes to the pthread library, but this is not without precedent. 

Scalability enhancements: There has been some discussion on LKML regarding private as well as NUMA-optimized hash tables. The futex hash table is shared across all processes and is protected by spinlocks which can lead to real overhead, especially on large systems. This overhead is not serving any useful purpose if these systems are partitioned on NUMA nodes, or even for processes that use private futexes exclusively. 

Futex test suite: Your author has been compiling a list of requirements for an exhaustive test suite to validate futex functionality. This test suite would serve as a regression suite for future development. The many corner cases and misuse cases possible with futexes complicate the test suite and present a challenge to its design. 

#### Acknowledgements

I would like to extend my thanks to John Stultz, Will Schmidt, Paul McKenney, Nivedita Singhvi, and, of course, Jon Corbet, whose reviews have made this article far more legible and complete than it would have been otherwise.
