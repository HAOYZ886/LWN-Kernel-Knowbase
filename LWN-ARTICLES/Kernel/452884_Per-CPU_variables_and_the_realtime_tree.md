---
title: "Per-CPU variables and the realtime tree"
url: https://lwn.net/Articles/452884/
date: "July 26, 2011"
category: "Per-CPU variables; Realtime"
author: "By Jonathan Corbet July 26, 2011"
---

By **Jonathan Corbet**  
July 26, 2011

One of the problems with relying on out-of-tree kernel code is that one can never be sure when that code might be updated for newer kernels. Keeping up with the kernel can be painful even for maintainers of small patches; it's much more so for those who maintain a large, invasive patch series. It is probably safe to say that, if the realtime preemption developers do not keep their patches current, there are very few other developers who are in a position to take on that work. So it was certainly discouraging for some realtime users to watch multiple kernel releases go by while the realtime patch series remained stuck at 2.6.33. 

The good news is that the roadblock has been overcome and there is now a new realtime tree for the 3.0 kernel. Even better news is that the realtime developers may have come up with a solution for one of the most vexing problems keeping the realtime code out of the mainline. The only potential down side is that this approach relies on an interesting assumption about how per-CPU data is used; this assumption will have to be verified with a lot of testing and, likely, a number of fixes throughout the kernel. 

Symmetric multiprocessing systems are nice in that they offer equal access to memory from all CPUs. But taking advantage of the feature is a guaranteed way to create a slow system. Shared data requires mutual exclusion to avoid concurrent access; that means locking and the associated bottlenecks. Even in the absence of lock contention, simply moving cache lines between CPUs can wreck performance. The key to performance on SMP systems is minimizing the sharing of data, so it is not surprising that a great deal of scalability work in the kernel depends on the use of per-CPU data. 

A per-CPU variable in the Linux kernel is actually an array with one instance of the variable for each processor. Each processor works with its own copy of the variable; this can be done with no locking, and with no worries about cache line bouncing. For example, some slab allocators maintain per-CPU lists of free objects and/or pages; these allow quick allocation and deallocation without the need for locking to exclude any other CPUs. Without these per-CPU lists, memory allocation would scale poorly as the number of processors grows. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Safe access to per-CPU data requires a couple of constraints, though: the thread working with the data cannot be preempted and it cannot be migrated while it manipulates per-CPU variables. If the thread is preempted, the thread that replaces it could try to work with the same variable; migration to another CPU could cause confusion for fairly obvious reasons. To avoid these hazards, access to per-CPU variables is normally bracketed with calls to `get_cpu_var()` and `put_cpu_var()`; the `get_cpu_var()` call, along with providing the address for the processor's version of the variable, disables preemption. So code which obtains a reference to a per-CPU data will not be scheduled out of the CPU until it releases that reference. Needless to say, any such code must be atomic. 

The conflict with realtime operation should be obvious: in the realtime world, anything that disables preemption is a possible source of unwanted latency. Realtime developers want the highest-priority process to run at all times; they have little patience for waiting while a low-priority thread gets around to releasing a per-CPU variable reference. In the past, this problem has been worked around by protecting per-CPU variables with spinlocks. These locks keep the code preemptable, but they wreck the scalability that per-CPU variables were created to provide and complicate the code. It has been clear for some time that a different solution would need to be found. 

With the [3.0-rc7-rt0](<https://lwn.net/Articles/452266/>) announcement, Thomas Gleixner noted that ""the number of sites which need to be patched is way too large and the resulting mess in the code is neither acceptable nor maintainable."" So he and Peter Zijlstra sat down to come up with a better solution for per-CPU data. The solution they came up with is surprisingly simple: whenever a process acquires a spinlock or obtains a CPU reference with `get_cpu()`, the scheduler will refrain from migrating that process to any other CPU. That process remains preemptable \- code holding spinlocks can be preempted in the realtime world - but it will not be moved to another processor. 

Disabling migration avoids one clear source of trouble: a process which is migrated in the middle of manipulating a per-CPU variable will end up working with the wrong CPU's instance of that variable. But what happens if a process is preempted by another process that needs to access the same variable? If preemption is no longer disabled, this unfortunate event seems like a distinct possibility. 

After puzzling over this problem for a bit, the path to enlightenment became clear: just ask Thomas what they are thinking with this change. What they are thinking, it turns out, is that any access to per-CPU data needs to be protected by some sort of lock. If need be, the lock itself can be per-CPU, so the locking need not reintroduce the cache line bouncing that the per-CPU variable is intended to prevent. In many cases, that locking is already there for other purposes. 

The realtime developers are making the bet that this locking is already there in almost every place where per-CPU data is manipulated, and that the exceptions are mostly for data like statistics used for debugging where an occasional error is not really a problem. When it comes to locking, though, a gut feeling that things are right is just not good enough; locking problems have a way of lurking undetected for long periods of time until some real damage can be done. Fortunately, this is a place where computers can help; the realtime tree will probably soon acquire an extension to the locking validator that checks for consistent locking around per-CPU data accesses. 

Lockdep is very good at finding subtle locking problems which are difficult or impossible to expose with ordinary testing. So, once this extension has been implemented and the resulting problem reports investigated and resolved, the assumption that all per-CPU accesses are protected by locking will be supportable. That process will likely take some time and, probably, a number of fixes to the mainline kernel. For example, there may well be bugs now where per-CPU variables are manipulated in interrupt handlers but non-interrupt code does not disable interrupts; the resulting race will be hard to hit, but possibly devastating when it happens. 

So, as has happened before, the realtime effort is likely to result in fixes which improve things for non-realtime users as well. Some churn will be involved, but, once it is done, there should be a couple of significant benefits: the realtime kernel will be more scalable on multiprocessor systems, and the realtime patches should be that much closer to being ready for merging into the mainline.
