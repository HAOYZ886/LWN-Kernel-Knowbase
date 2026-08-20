---
title: "Improving C-library scalability with restartable sequences"
url: https://lwn.net/Articles/946870/
date: "October 17, 2023"
category: Restartable sequences
author: "By Jonathan Corbet October 17, 2023 Cauldron"
---

By **Jonathan Corbet**  
October 17, 2023

* * *

[Cauldron](<https://lwn.net/Archives/ConferenceByYear/#2023-GNU_Tools_Cauldron>)

The Linux kernel has supported [restartable sequences](<https://lwn.net/Articles/697979/>) (sometimes referred to as "RSEQ") since 2018, but it remains a bit of a niche feature, mostly useful to performance-oriented developers who do not mind writing assembly code. According to Mathieu Desnoyers, the developer behind the kernel's implementation of restartable sequences, this feature can be applicable to a much wider range of performance-sensitive code with proper library support. He came to the [2023 GNU Tools Cauldron](<https://gcc.gnu.org/wiki/cauldron2023>) to present the case for use of restartable sequences within the GNU C Library (glibc). 

There are, he began, a number of approaches that are used to improve the scalability of user-space code; most of them revolve around partitioning the workload in one way or another. Use of thread-local storage to minimize contention for shared data is one example. Applications can also use [read-copy-update](<https://lwn.net/Articles/573424/>), hazard pointers, or reference counting (which works best in the absence of frequent changes, he said). Another approach is per-CPU data structures; they are heavily used in the kernel, he said, but can be made to work in user space as well. The kernel can rely on techniques like disabling preemption to guarantee exclusive access to a per-CPU data structure, but user space has no such luxury. That is where restartable sequences can help. 

[![\[Mathieu
Desnoyers\]](https://static.lwn.net/images/conf/2023/gtc/MathieuDesnoyers-sm.png)](<https://lwn.net/Articles/946897/>) The best approach for any given situation will depend heavily on the workload and its data-access patterns. The choice should be based on metrics — benchmarks, profiles, tracing, and the like — that clearly show where the scalability bottlenecks lie and how a given technique improves the situation. He stressed that the ideas he was presenting did not have such a firm foundation; they were mostly based on code review, and would need to prove their value before any work in that direction goes far. 

Restartable sequences, he said, were added to the kernel in the 4.18 release. The feature has been used within glibc to implement [`sched_getcpu()`](<https://man7.org/linux/man-pages/man3/sched_getcpu.3.html>), since it makes the current CPU number available to every thread. Code using restartable sequences starts by sharing a structure indicating an address range delineating a critical section, along with an abort address, with the kernel. In normal execution, the code in the critical section will run and commit its work with a single atomic store instruction at the end. If the thread is preempted while executing the critical section, though, it will have lost its exclusive access to the per-CPU data structure it was working with and cannot safely continue; in that case, it will be made to jump to the abort address, from which it can restart the operation. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The 6.3 kernel added a NUMA ID field to the data shared between threads and the kernel, allowing glibc's implementation of [`getcpu()`](<https://man7.org/linux/man-pages/man2/getcpu.2.html>) to be optimized. This release also added the concept of per-memory-map concurrency using [virtual CPU numbers](<https://lwn.net/Articles/885818/>), allowing more efficient use of per-CPU data structures. A new, extensible API was added as well, but support for it has not yet landed in glibc; that will need to happen soon, he said, since there is little room left for expansion in the older API. 

The [librseq library](<https://git.kernel.org/pub/scm/libs/librseq/librseq.git>) is being developed to make it easier for developers to take advantage of restartable sequences. It implements a number of common data structures, including per-CPU counters, linked lists, and spinlocks, along with a number of low-level atomic operations. There is support for several architectures. This library is still in an early stage of development; there has been no proper release yet. 

The implementation of per-CPU counters follows the usual pattern: a separate counter is maintained on each CPU and can be incremented without contention. The total value of the counter is obtained by adding up each of the per-CPU values. This algorithm is good, he said, in situations where counters are frequently updated but seldom read. There is, however, the problem of how to access "remote data" (counters on CPUs other than the current one) safely to get a precise sum or make changes to the counter array as a whole. 

The answer turns out to be an extension to the [expedited `membarrier()` system call](<https://lwn.net/Articles/728795/>). The `MEMBARRIER_CMD_PRIVATE_EXPEDITED_RSEQ` option was [added](<https://git.kernel.org/linus/2a36ab717e8f>) to the 5.10 kernel; it executes a memory barrier but also aborts any currently running RSEQ critical sections; Desnoyers called this operation an "RSEQ fence". For a data structure like a set of per-CPU counters, a manager thread can replace the entire structure, then issue the RSEQ fence. After that, no threads will be working with the older structure, and the manager will have exclusive access to it. 

There is an implementation of a per-CPU spinlock that is intended to be fast when accessed only by the local CPU; it can be taken remotely in a slower path. Each CPU's spinlock contains a bit describing how the lock is to be acquired; most of the time the bit is clear, indicating that no thread is trying to gain remote access. In this case, a thread can obtain the lock by reading its value, checking that it is free, and claiming it by setting its value to locked — all within an RSEQ critical section, of course. When a remote thread needs to take the lock, it begins by setting the remote-access bit in the lock, followed by an RSEQ fence. When that bit is set, an atomic compare-and-swap is required to obtain the lock; at that point, the local and remote thread can contend for it in the traditional (slower) way. 

Memory allocation was one of the original use cases for restartable sequences. One approach is to use per-CPU free lists, indexed with the virtual CPU number. Additions to and removals from the list can be done with fast push and pop operations within an RSEQ critical section. If a slow path is needed for some operations, an RSEQ fence or per-CPU spinlock can be used to gain the needed access. 

There are a number of locks within glibc, he said, that could be optimized with either RSEQ or RCU; these include the dynamic loader lock, the dynamic loader stack cache lock, the default pthread attribute lock, gettext locks, and the timezone lock. The last one is acquired frequently, since it is needed by functions like `localtime()`, but the time zone changes infrequently (if at all). An RSEQ-based update mechanism could help here, he said. 

POSIX condition variables are a harder problem; they use a mutex internally to serialize operations and can become a contention point when waits and wakeups are frequent. That is inherent in the design of POSIX threads, though, and difficult to change. As an alternative, he has created an API called `urcu_wait()`, implemented in [liburcu](<https://liburcu.org/>). It implements a stack of waiting threads that can be accessed quickly, and falls back to futexes when the need arises. 

At this point, time was running out. Desnoyers quickly mentioned the [adaptive spinlock implementation](<https://lwn.net/Articles/944895/>) that he is working on with André Almeida. With a small extension to the RSEQ API, user-space code is able to tell if another thread is currently running and decide whether it should actively wait for a lock to become free. 

Overall, there was interest in the ideas presented there; glibc already has [some support for restartable sequences](<https://lwn.net/Articles/883104/>), so there should be no real impediment to using the feature more extensively internally. It is all, as they say, just a matter of writing the code and verifying that it truly improves the situation.
