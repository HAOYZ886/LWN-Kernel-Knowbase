---
title: Reimplementing printk()
url: https://lwn.net/Articles/780556/
date: "February 26, 2019"
category: Kernel messages; printk
author: "By Jake Edge February 26, 2019"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
February 26, 2019

The venerable `printk()` function has been part of Linux since the very beginning, though it has undergone a fair number of changes along the way. Now, John Ogness is proposing to fundamentally rework `printk()` in order to get rid of handful of issues that currently plague it. The proposed code does this by adding yet another ring-buffer implementation to the kernel; this one is aimed at making `printk()` work better from hard-to-handle contexts. For a task that seems conceptually simple—printing messages to the console—`printk()` is actually a rather complex beast; that won't change if these patches are merged, though many of the problems with the current implementation will be removed. 

In the [cover letter](<https://lwn.net/ml/linux-kernel/20190212143003.48446-1-john.ogness@linutronix.de/>) of his RFC patch set, Ogness lays out seven problems that he sees with the current `printk()` implementation. The buffer used by `printk()` is protected by a raw spinlock, which restricts the contexts from which the buffer can be accessed. Calling `printk()` from a non-maskable interrupt (NMI) or a recursive context, where something called from `printk()` also calls `printk()`, currently means that the logging of the message is deferred, which could cause the message to be lost. Printing to slow consoles can result in large latencies, a `printk()` call may end up taking unbounded time while other deferred messages are printed ahead of the one the caller actually wanted to print. 

Two other problems are identified by Ogness. Timestamps on the messages are added when a message is added to the buffer but, due to deferrals, that can happen well after the `printk()` call was made. While that behavior has the side effect of nicely sorting the messages in terms of time, it is ""neither accurate nor reliable"". In addition, because `printk()` tries to satisfy all of its users, it ends up compromising too much: 

Loglevel INFO is handled the same as ERR. There seems to be an endless effort to get printk to show _all_ messages as quickly as possible in case of a panic (i.e. printing from any context), but at the same time try not to have printk be too intrusive for the callers. These are conflicting requirements that lead to a printk implementation that does a sub-optimal job of satisfying both sides. 

In order to fix those problems, he is proposing the addition of a kernel-internal `printk()` ring buffer that allows for multiple lockless readers; writers use a per-CPU synchronization mechanism that works in any context. This ring-buffer implementation was inspired by a [suggestion from Peter Zijlstra](<https://lwn.net/ml/linux-kernel/20181017140044.GK3121%40hirez.programming.kicks-ass.net/>), Ogness said. The actual writing of the messages is moved to a dedicated kernel thread, which is preemptible, so the latencies can be bounded. The new ring buffer will be allocated in the initialized data segment of the kernel, so it will be available early in the boot process, even before the memory-management subsystem is available. Timestamps will be generated early in the `printk()` function. 

Beyond that, a new kind of "emergency message" is defined. Those messages will appear on certain consoles immediately; no waiting for any other `printk()` messages that are queued up for the kernel thread. In order to participate, consoles will need to implement the new `write_atomic()` operation. The patch set includes an [implementation of `write_atomic()` for the 8250 UART driver](<https://lwn.net/ml/linux-kernel/20190212143003.48446-21-john.ogness@linutronix.de/>). There is a new kernel configuration parameter, `CONFIG_LOGLEVEL_EMERGENCY`, which sets the lowest log-level value for emergency messages; it defaults to `LOG_WARNING`. That value can also be set by the "`emergency_loglevel=`" kernel command-line parameter at boot time. 

So instead of trying to ensure that all messages go out as quickly as possible, the proposal would effectively partition `printk()` messages into two buckets—at least for consoles that implement `write_atomic()`. Regular messages will be written out by the kernel thread, which gets scheduled and preempted normally, so it may not be flushing all of the messages right away. The messages at the emergency level and above will go out immediately to an emergency console. As he [noted](<https://lwn.net/ml/linux-kernel/87d0nv248b.fsf@linutronix.de/>) in the thread, there are options if losing regular messages in a crash becomes a problem: 

As long as all critical messages are print directly and immediately to an emergency console, why is it is problem if the informational messages to consoles are sometimes delayed or lost? And if those informational messages _are_ so important, there are things the user can do. For example, create a realtime userspace task to read /dev/kmsg. 

There are some downsides and open issues in the proposal, which Ogness also lists. The output from `printk()` has changed somewhat, which may have unforeseen consequences. A CPU ID will be printed as part of emergency messages to help disambiguate multiple simultaneous messages; those messages are separated from regular `printk()` messages using a newline character, though there may be an option added to make them stand out more. In addition: 

Be aware that printk output is no longer time-sorted. Actually, it never was, but now you see the real timestamps. This seems strange at first. 

More details on the ring buffer and its API, along with some early performance numbers, can be seen in a [patch](<https://lwn.net/ml/linux-kernel/20190212143003.48446-2-john.ogness@linutronix.de/>) adding a file to the `Documentation` directory.

The reaction to the patch set has been positive overall; there have been the usual questions and comments, of course. `printk()` has been the subject of two relatively recent Kernel Summit sessions ([in 2016](<https://lwn.net/Articles/705938/>) and [2017](<https://lwn.net/Articles/737822/>)); two of those proposing `printk()` changes at the summits, Sergey Senozhatsky and Petr Mladek, have also commented on the patches. Mladek had suggestions for improvements to the ring-buffer code, [as did Linus Torvalds](<https://lwn.net/ml/linux-kernel/CAHk-=whToBNHN_X_BQkO4zSyhSBaGqTH8XEtxg2_AxoBdDnEdg@mail.gmail.com/>). There have been no major complaints about the overall goal and plan, however, so it would seem that we could see these changes go into the mainline in a development cycle or two.
