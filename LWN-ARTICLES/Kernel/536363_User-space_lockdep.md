---
title: "User-space lockdep"
url: https://lwn.net/Articles/536363/
date: "February 6, 2013"
category: Lockdep
author: "By Jonathan Corbet February 6, 2013"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 6, 2013

The kernel's [locking validator](<https://lwn.net/Articles/185666/>) (often known as "lockdep") is one of the community's most useful pro-active debugging tools. Since its introduction in 2006, it has eliminated most deadlock-causing bugs from the system. Given that deadlocks can be extremely difficult to reproduce and diagnose, the result is a far more reliable kernel and happier users. There is a shortage of equivalent tools for user-space programming, despite the fact that deadlock issues can happen there as well. As it happens, making lockdep available in user space may be far easier than almost anybody might have thought. 

Lockdep works by adding wrappers around the locking calls in the kernel. Every time a specific type of lock is taken or released, that fact is noted, along with ancillary details like whether the processor was servicing an interrupt at the time. Lockdep also notes which other locks were already held when the new lock is taken; that is the key to much of the checking that lockdep is able to perform. 

To illustrate this point, imagine that two threads each need to acquire two locks, called _A_ and _B_ : 

> ![\[Cheesy lock diagram\]](https://static.lwn.net/images/2013/lockdep/lockdep1.png)

If one thread acquires _A_ first while the other grabs _B_ first, the situation might look something like this: 

> ![\[Cheesy lock diagram\]](https://static.lwn.net/images/2013/lockdep/lockdep2.png)

Now, when each thread goes for the lock it lacks, the system is in trouble: 

> ![\[Cheesy lock diagram\]](https://static.lwn.net/images/2013/lockdep/lockdep3.png)

Each thread will now wait forever for the other to release the lock it holds; the system is now deadlocked. Things may not come to this point often at all; this deadlock requires each thread to acquire its lock at exactly the wrong time. But, with computers, even highly unlikely events will come to pass sooner or later, usually at a highly inopportune time. 

This situation can be avoided: if both threads adhere to a rule stating that _A_ must always be acquired before _B_ , this particular deadlock (called an "AB-BA deadlock" for obvious reasons) cannot happen. But, in a system with a large number of locks, it is not always clear what the rules for locking are, much less that they are consistently followed. Mistakes are easy to make. That is where lockdep comes in: by tracking the order of lock acquisition, lockdep can raise the alarm anytime it sees a thread acquire _A_ while already holding _B_. No actual deadlock is required to get a "splat" (a report of a locking problem) out of lockdep, meaning that even highly unlikely deadlock situations can be found before they ruin somebody's day. There is no need to wait for that one time when the timing is exactly wrong to see that there is a problem. 

Lockdep is able to detect more complicated deadlock scenarios than the one described above. It can also detect related problems, such as locks that are not interrupt-safe being acquired in interrupt context. As one might expect, running a kernel with lockdep enabled tends to slow things down considerably; it is not an option that one would enable on a production system. But enough developers test with lockdep enabled that most problems are found before they make their way into a stable kernel release. As a result, reports of deadlocks on deployed systems are now quite rare. 

Kernel-based tools often do not move readily to user space; the kernel's programming environment differs markedly from a normal C environment, so kernel code can normally only be expected to run in the kernel itself. In this case, though, Sasha Levin noticed that there is not much in the lockdep subsystem that is truly kernel-specific. Lockdep collects data and builds graphs describing observed lock acquisition patterns; it is code that could be run in a non-kernel context relatively easily. So Sasha proceeded to put together [a patch set](<https://lwn.net/Articles/536366/>) creating a lockdep library that is available to programs in user space. 

Lockdep does, naturally, call a number of kernel functions, so a big part of Sasha's patch set is a long list of stub implementations shorting out calls to functions like `local_irq_enable()` that have no meaning in user space. An abbreviated version of `struct task_struct` is provided to track threads in user space, and functions like `print_stack_trace()` are substituted with user-space equivalents (`backtrace_symbols_fd()` in this case). The kernel's internal (used by lockdep) locks are reimplemented using POSIX thread ("pthread") mutexes. Stub versions of the include files used by the lockdep code are provided in a special directory. And so on. Once all that is done, the lockdep code can be built directly out of the kernel tree and turned into a library. 

User-space code wanting to take advantage of the lockdep library needs to start by including `<liblockdep/mutex.h>`, which, among other things, adds a set of wrappers around the `pthread_mutex_t` and `pthread_rwlock_t` types and the functions that work with them. A call to `liblockdep_init()` is required; each thread should also make a call to `liblockdep_set_thread()` to set up information for any problem reports. That is about all that is required; programs that are instrumented in this way will have their pthreads mutex and rwlock usage checked by lockdep. 

As a proof of concept, the patch adds instrumentation to the (thread-based) `perf` tool contained within the kernel source tree. 

One of the key aspects of Sasha's patch is that it requires no changes to the in-kernel lockdep code at all. The user-space lockdep library can be built directly out of the kernel tree. Among other things, that means that any future lockdep fixes and enhancements will automatically become available to user space with no additional effort required on the kernel developers' part. 

In summary, this patch looks like a significant win for everybody involved; it is thus not surprising that opposition to its inclusion has been hard to find. There has been [a call](<https://lwn.net/Articles/536373/>) for some better documentation, explicit mention that the resulting user-space library is GPL-licensed, and a runtime toggle for lock validation (so that the library could be built into applications but not actually track locking unless requested). Such details should not be hard to fill in, though. So, with luck, user space should have access to lockdep in the near future, resulting in more reliable lock usage.
