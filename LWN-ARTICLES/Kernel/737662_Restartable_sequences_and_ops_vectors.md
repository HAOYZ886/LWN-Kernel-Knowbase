---
title: Restartable sequences and ops vectors
url: https://lwn.net/Articles/737662/
date: "October 31, 2017"
category: Restartable sequences
author: "By Jonathan Corbet October 31, 2017 2017 Kernel Summit"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 31, 2017

* * *

[2017 Kernel Summit](<https://lwn.net/Articles/KernelSummit2017/>)

Some technologies find their way into the kernel almost immediately; others need to go through multiple iterations over a number of years first. Restartable sequences, a mechanism for lockless concurrency control in user space, fall into the latter category. At the 2017 Kernel Summit, Mathieu Desnoyers discussed yet another implementation of this concept — but this one may not be the last word either. 

The [core idea behind restartable sequences](<https://lwn.net/Articles/697979/>) has not changed. An application defines a special region of code that, it is hoped, will run without interruption. This code performs some operation of interest on a per-CPU data structure that can be committed with a single instruction at the end. For example, it may prepare to remove an item from a list, with the final instruction setting a pointer that actually effects this change and makes it visible to other threads running on the same CPU. If the thread is preempted in the middle of this work, it may contend with another thread working on the same data structure. In this case, the kernel will cause the thread to jump to an abort sequence once it runs again; the thread can then clean up and try again (the "restart" part of the name). Most of the time, though, preemption does not happen, and the restartable sequence will implement a per-CPU, atomic operation at high speed. 

Restartable sequences have been around for some time and are evidently in use at Google and elsewhere. But there are some rough edges, one of which is debugging. Single-stepping through a restartable sequence with a debugger will be an exercise in frustration, since the sequence will never run uninterrupted and, thus, always abort. Fixing this problem requires the implementation of some way to execute the sequence as a single step. 

The solution in [the current patch set](<https://lwn.net/Articles/736316/>) is a new system call: 

```
int cpu_opv(struct cpu_op *ops, int opcount, int cpu, int flags);
```

The purpose of this system call is to accept a sequence of operations (an "ops vector") and execute it atomically. Each entry in the `ops` array is a single operation; the array has a maximum length of sixteen operations. The available operations include comparisons, memory copies, and basic arithmetic. The amount of data that can be operated on is bounded (to limit the maximum execution time of the vector), and all of that data is locked into memory before the execution of the ops vector begins. The vector is run in the processor indicated by `cpu`; the `flags` field must be zero in the current implementation. 

The ops vector is meant to be used as a fallback when a restartable sequence aborts; it can be run during single-stepping or any other situation where the sequence itself is unable to complete successfully (it is not a suitable replacement for the sequence entirely; as a system call, it will be quite a bit slower). Users of restartable sequences would thus need to create a second [![\[Mathieu
Desnoyers\]](https://static.lwn.net/images/conf/2017/osse/MathieuDesnoyers-sm.jpg)](<https://lwn.net/Articles/737665/>) implementation of their algorithm in this new language and run it when the original sequence fails. This idea, Desnoyers said, came to him in the shower one day. It is, he said, a relatively simple solution to the problem. 

This was the point where your editor was unable to resist raising his hand and asking whether, rather than adding yet another interpreter to the kernel, Desnoyers could use the existing BPF language and interpreter. The existing BPF verifier could likely be adapted to the needs of the ops-vector mechanism. Desnoyers replied that BPF carries a lot of weight that is not needed here and, in any case, the ops vector should almost never actually run in real-world use. But then he went on to say that ops vectors could also be employed for simple housekeeping tasks that may not need a full restartable-sequence implementation. 

Andy Lutomirski jumped in to say that BPF seemed like a reasonable solution to the problem; the BPF interpreter's context mechanism could be used to manage operands to the vector, for example. Peter Zijlstra pointed out that BPF programs have a large kernel-space context associated with them; a program might have one-hundred restartable sequences, which would add up to a lot of overhead. 

Lutomirski then said that he has his own version of restartable sequences that he has been working on. Rather than abort when preemption occurs, it aborts when an actual data conflict happens. Single-stepping works in this implementation, he said. Desnoyers replied that such an approach would make the implementation more complex, but Lutomirski said that it is still better than requiring every user to implement their algorithms twice. The slow path will be poorly tested at best, and developers will often get it wrong, he said. Zijlstra replied that there would be a library that would take care of the details for most uses, though, and Ben Herrenschmidt said that only developers who truly care about restartable sequences will deal with things at that level. 

Desnoyers moved on to the use cases for restartable sequences — an important topic, since Linus Torvalds has [made it clear](<https://lwn.net/Articles/737664/>) that he will not merge this code without clear evidence that it will be used. The LTTng tracing code can use this feature for fast user-space tracing across processes, Desnoyers said; he would also like it for his user-space read-copy-update implementation. The jemalloc and GNU C Library `malloc()` implementations can speed things up with restartable sequences. There is a use case for per-CPU statistics counters. Matthew Wilcox added that the developers of the DPDK user-space driver system also want this mechanism. Herrenschmidt said that, in the end, all of the concurrency issues that apply to the kernel also apply to user space. 

The final part of the discussion wandered over various topics, including the details of how multiple, independent users can share a restartable-sequences region and whether maybe the classic BPF interpreter might be a better tool for the ops-vector job than extended BPF. Desnoyers said that he would look into the BPF option; expect the conversation to continue on the mailing lists. 

[Your editor would like to thank the Linux Foundation, LWN's travel sponsor, for supporting his travel to this event].
