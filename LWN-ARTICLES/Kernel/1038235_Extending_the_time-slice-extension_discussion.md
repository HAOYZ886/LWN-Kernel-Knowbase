---
title: "Extending the time-slice-extension discussion"
url: https://lwn.net/Articles/1038235/
date: "September 18, 2025"
category: "Releases-7.0; Scheduler-Time-slice extension"
author: "By Jonathan Corbet September 18, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
September 18, 2025

Time-slice extension is a proposed scheduler feature that would allow a user-space process to request to not be preempted for a short period while it executes a critical section. It is an idea that has been circulating for years, but efforts to implement it [became more serious](<https://lwn.net/Articles/1009509/>) in February of this year. The latest developer to make an attempt at time-slice extension is Thomas Gleixner, who has posted [a new patch set](<https://lwn.net/ml/all/20250908225709.144709889@linutronix.de>) with a reworked API. Chances are good that this implementation is close to what will actually be adopted by the kernel. 

Imagine a user-space thread that holds a (user-space) spinlock; if that thread is preempted by another thread which subsequently attempts to acquire that same lock, the preempting thread will spin indefinitely, having displaced the thread that could release the lock. That is not a path toward optimal performance. If, though, the lock-holding thread could ask to not be preempted while holding the lock, this scenario could be avoided; time-slice extension is meant to at least give that thread a chance to run long enough to finish its work before being kicked out of the CPU. 

The February implementation of this work was by Steve Rostedt; after that work bogged down, an implementation by Prakash Sangappa [was considered](<https://lwn.net/Articles/1029093/>) for a while, but also ran into disagreements over its API and implementation details. Gleixner's work is an attempt to create a version that will pass muster. To get there, he first had to [rework the restartable sequences implementation](<https://lwn.net/Articles/1033955/>) to address a number of problems he found there; [a new version of that series](<https://lwn.net/ml/all/20250908212737.353775467@linutronix.de>) was posted on September 8\. 

In this version of the time-slice-extension API, a process must explicitly enable the feature before attempting to use it. That is done with a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) command named `PR_RSEQ_SLICE_EXTENSION_SET`. This requirement serves two purposes: it allows the kernel to avoid some overhead for the (presumed) majority of processes that do not use time-slice extension, and a process that _can_ use the feature will get an indication of whether the kernel actually supports it. There is also a new `PR_RSEQ_SLICE_EXTENSION_GET` command to query whether the feature is currently enabled for the calling process. 

Like most of its predecessors, Gleixner's implementation uses the restartable sequences infrastructure, including its block of memory shared between user space and the kernel, in the implementation of the new feature. This time around, though, the [`rseq` structure](<https://elixir.bootlin.com/linux/v6.16.7/source/include/uapi/linux/rseq.h#L56>) stored in that block is given a new field: 

```
__u32 slice_ctrl;
```

To request a time-slice extension, a thread sets the `RSEQ_SLICE_EXT_REQUEST` bit in that field; a simple assignment is sufficient, there is no need for any special atomic operations. If that thread is then interrupted, the kernel will handle whatever task caused the interruption; before returning to user space, the kernel checks whether anything has happened that would normally cause the current thread to be scheduled out. If the thread is marked for rescheduling, it would normally lose access to the CPU; if, however, the request bit is set in `slice_ctrl`, the conditions are right, and the kernel is in a good mood, it will consider giving that thread just a bit more time before the eviction happens. 

**About that timer** : the setting of the 30µs high-resolution timer is one of the most expensive parts of implementing this feature, since it requires reprogramming the hardware. The kernel reduces that overhead by looking at the amount of time left in the thread's time slice; if it exceeds the grant period, the timer will not be set. 

Whether the conditions are right depends on a few things, including the reason why rescheduling the process is called for; the kernel is unlikely to grant an extension if a high-priority realtime process needs the CPU, for example. Whether or not the extension is allowed, the kernel will clear the `RSEQ_SLICE_EXT_REQUEST` bit in `slice_ctrl` to inform the thread that it was interrupted. If the kernel does decide to grant the extension, it will, before returning to user space, set the `RSEQ_SLICE_EXT_GRANTED` bit in `slice_ctrl` and set a timer for (by default) 30µs. 

The user-space thread should run its critical section and, at completion, check the request bit with an atomic test-and-clear operation. If that bit had already been cleared by the kernel, the thread will know that an interrupt has happened. In that case, it must check for `RSEQ_SLICE_EXT_GRANTED` to see whether it is running on borrowed time; if so, it must immediately call the new `rseq_slice_yield()` system call to tell the kernel that the critical section is complete. That call is likely to result in the rescheduling of the thread. 

There are a few things that a potential user of this feature should be aware of. One is that the extension grant can be revoked if the kernel changes its mind; that will happen unconditionally if the thread is interrupted again while running under the extension grant. In that case, the `RSEQ_SLICE_EXT_GRANTED` bit will be cleared, and the thread does not have to call `rseq_slice_yield()`. If the thread runs its critical section to completion, though, some care must be taken to not run afoul of the kernel's rules. The `rseq_slice_yield()` call is not optional; if the kernel's 30µs timer expires before that call is made, the grant will be ended and the thread rescheduled. A more severe fate awaits any thread that makes any system call other than `rseq_slice_yield()` while running with an extension grant; that thread will be terminated immediately. 

Rostedt's February version only worked if the scheduler was running in the [lazy preemption](<https://lwn.net/Articles/994322/>) mode; that limitation was controversial at the time. Gleixner's implementation does not have that limitation and, in fact, will explicitly not work as well when lazy preemption is in use, especially on realtime kernels, where preemption can happen anywhere in the kernel and not just at the point of return to user space. Implementing time-slice extension in that environment would add to the overhead of the feature, whether it is in use or not. In the earlier discussions, there had been talk of making time-slice extension available to realtime processes as well, but that is not a part of this series. 

The API for this feature may not yet be entirely set in stone. Mathieu Desnoyers had [a couple of requests](<https://lwn.net/ml/all/a65dfd2c-b435-4d83-89d0-abc8002db7c7@efficios.com>), the first of which was to allow the user-space critical sections to be nested, so that a second critical section could be entered while already running within a critical section. That would require turning the `RSEQ_SLICE_EXT_REQUEST` into a counter informing the kernel of just how many nested critical sections were being executed at any given time. Gleixner [pondered the request](<https://lwn.net/ml/all/874it6qzd0.ffs@tglx>) for a bit before concluding that nesting should be implemented in user space, if at all. 

The second request was to allow _any_ system call to signal an end to the critical section, rather than killing the thread. Otherwise, he said, the feature would see far fewer users: ""Handling syscall within granted extension by killing the process will likely reserve this feature to the niche use-cases"". Gleixner rejected that reasoning, saying: ""Having this used only by people who actually know what they are doing is actually the preferred outcome"". Even so, he had previously [said](<https://lwn.net/ml/all/87a52zr5sv.ffs@tglx>) that he would be willing to consider allowing any system call as a way to end the critical section, but that it would require changing the system-call entry code in a way that he feared might not be acceptable to the scheduler developers (who have not yet expressed an opinion on the subject). 

While there may be discussion about the details for a while, it seems likely that the eventual API for time-slice extension will be fairly close to what has been described here. The feature could perhaps be ready to merging within a couple of development cycles. This being the kernel community, though, there is always a chance that somebody will make a request to extend the discussion beyond its natural end.
