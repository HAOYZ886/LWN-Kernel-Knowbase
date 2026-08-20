---
title: "What's in the realtime tree"
url: https://lwn.net/Articles/252716/
date: "October 3, 2007"
category: Realtime
author: "By Jonathan Corbet October 3, 2007"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
October 3, 2007

It has been almost exactly three years since Sven-Thorsten Dietrich posted [a set of realtime improvements](<http://lwn.net/Articles/105866/>) to the linux-kernel list. That particular body of code was upstaged by the realtime preemption work done by Ingo Molnar and others, but it deserves some credit for kicking off a development effort which continues to this day. After three years, many parts of the realtime preemption patch set have been merged into the mainline kernel, including dynamic tick support, a rewritten interrupt subsystem, mutexes, priority inheritance, high-resolution timers, and more. At this point, we are all running kernels which have benefited from the realtime preemption project. 

The job of merging the realtime preemption work into the mainline is not complete, though. Indeed, a look at the [2.6.23-rc8-rt1](<http://lwn.net/Articles/251889/>) tree announcement shows that there are nearly 400 individual patches sitting there. This seems like a good opportunity to have a look at the realtime tree and see what remains to be merged. 

The core of this patch set remains the realtime mutex code. When the kernel is configured for realtime operation, a bunch of (improved, but still scary) preprocessor magic causes normal spinlocks to be replaced by realtime mutexes, which have different properties. In particular, realtime mutexes are fully preemptible and have priority inheritance. With most kernel spinlocks replaced by these mutexes, there are few places in the kernel which are able to cause arbitrary latencies. 

Merging realtime mutexes should, in theory, not be a problem; they are not actually used unless explicitly configured into the kernel, and it is assumed that most users will not configure things that way. Such a fundamental change to a core mutual exclusion mechanism will always raise eyebrows, though. So there have been no attempts to merge this code so far, and it is likely that most of the other parts of the realtime tree will find their way into the mainline first. 

Code which could go in sooner is the threaded interrupt handlers patch. Putting interrupt handlers into threads allows them to be scheduled along with most other system activities and eliminates another potential source of latency. It also can improve the robustness of the system and make it easier to find bugs in interrupt code. So this patch could be merged and possibly even made the default - though there will always be a small number of interrupt handlers which must be run directly. 

Also in the realtime tree is a patch which moves all software interrupt processing into dedicated threads. Then there is a patch which allows hardware interrupt handling threads to process any pending software interrupts before yielding the processor. This optimization avoids the need for a context switch to a separate software IRQ thread to get those interrupts delivered. 

One of the sticking points with the realtime patches has been their interaction with the read-copy-update mechanism. The current code requires that preemption be disabled while references to RCU-protected data structures exist, but disabling preemption ruins the guarantees that the realtime code is trying to provide. The answer is a somewhat more complicated, preemptible RCU implementation. With luck, LWN will have an article on how preemptible RCU works in the near future. 

Nick Piggin's lockless pagecache patches have found their way into the realtime tree. These patches make a number of low-level changes to the memory management and radix tree code so that some pagecache operations can be done without taking any locks. This code has been in circulation for some time without making it into the mainline, but it seems like a win in a number of scalability situations. Another patch (by Peter Zijlstra) gets rid of the locking in the `kmap()` code, improving latencies in systems using high memory. 

The wisdom of allowing Java programs to mess with physical memory is not a topic which should need further discussion here.  Another patch which has been out of the mainline for quite some time - and likely to remain that way - is Ted Ts'o's `/dev/rmem` facility. This device allows direct access to physical memory - a feature which is required on any system which wants to pass the realtime Java conformance tests. The wisdom of allowing Java programs to mess with physical memory is not a topic which should need further discussion here. 

The realtime tree contains an extensive set of tools for tracking down parts of the kernel which cause excessive latencies. This code has, over the years, been put to good use in identifying and breaking up kernel code which hogs the processor unnecessarily. These patches would seem like a good match for the mainline, especially given recent discussions on the value of adding more instrumentation to the kernel. The first step in solving problems is being able to measure them. 

For reasons which are unclear to your editor, the realtime tree contains the venerable [realtime security module](<http://lwn.net/Articles/118785/>), which was definitively refused entry into the mainline a few years ago. The module is marked as being obsolete - but it is still there. 

Quite a few other changes can be found in this tree. The SLUB allocator is not an option for realtime kernels. Instead, this tree uses a modified version of the slab allocator which replaces interrupt-based single-CPU locking with a set of specific per-CPU locks. The global `files_lock` has been removed in favor of tightly-locked per-CPU lists. To help with the creation of such lists, a new locked-list type has been added. The tasklet code has been reworked for better threading, but the [tasklet elimination patch](<http://lwn.net/Articles/239633/>) is not present. There's also quite a few architecture-specific patches adding various features (such as clockevents) needed by the realtime tree and fixing problems. 

All told, there is a lot of code sitting in the realtime tree despite all of the mainline merging which has happened over the last couple of years or so. The stated plan is to merge most of this code in the not-too-distant future, but it is not clear when that will happen. In particular, some of the core realtime developers are likely to be severely distracted by the i386/x86_64 merger during the 2.6.24 cycle, so they may not manage to move much of the realtime code toward the mainline.
