---
title: The beginning of the realtime preemption debate
url: https://lwn.net/Articles/138174/
date: "June 1, 2005"
category: Latency; Preemption; Realtime
author: 
---

Merging Ingo Molnar's realtime preemption work was never going to be a quiet process. The noise has, in fact, begun long before Ingo has even proposed his work for inclusion. Now might be a good time to catch up with the debate as a way of seeing how the arguments might go in the future. 

The realtime preemption patches attempt to provide a guaranteed maximum response time for high-priority user-space processes - just like a "real" realtime operating system would. This goal is achieved by making _everything_ in the kernel preemptible. No matter what the kernel is doing on a given processor, if a higher-priority process becomes runnable, it will be scheduled immediately. Many changes are required to make the whole kernel preemptible; the core parts are: 

  * New locking primitives. The spinlocks used by the kernel can cause any number of processors to stall while waiting for a lock to become free. Code which holds a spinlock cannot be preempted, or a deadlocked kernel could result. The realtime preemption patches introduce a new mutual exclusion type (the rt_mutex) which does not spin, and, thus, will not stall a processor. The spinlocks and semaphores currently used in the kernel are all converted over to the new rt_mutex type, and all code which runs with spinlocks held becomes preemptible. The rt_mutex type also implements priority inheritance, so that a low-priority process will not block a higher-priority process (for long, at least) by losing the processor while holding an important lock. 

  * Threaded interrupt handlers. Interrupt handlers can create latencies by monopolizing the processor for long periods of time. The realtime preemption patch moves interrupt handling into kernel threads, which contend for the processor with all other processes in the system. If a certain realtime task is more important than interrupt handling, its priority can be set accordingly. 

  * Various other mutual exclusion mechanisms, including read-copy-update, per-CPU variables, and seqlocks, require that preemption be disabled. All of these mechanisms are changed for the realtime preemption mode, usually by making them look more like regular spinlocks. 

The realtime preemption patch set (at version [-RT-2.6.12-rc5-V0.7.47-10](<https://lwn.net/Articles/137796/>) as of this writing) is clearly large and intrusive - it would be hard to make fundamental changes like those listed above any other way. It should be noted that Ingo has gone out of his way to minimize this intrusiveness, however: the patch is written to minimize code changes, and the kernel functions as always if realtime preemption is not selected at configuration time. The merging of this patch set would not force the new preemption model on users. 

[According to Lee Revell](<https://lwn.net/Articles/138180/>), the realtime preemption patches are already seeing some serious use: 

All of the Linux audio oriented distributions are already shipping -RT kernels, and most of the serious Linux audio users who use general purpose distros are running it. That's a few thousand people running it 24/7 for months, and it's been at least a month since any of these users found a real bug in -RT. 

Certainly the discussions that inevitably follow the release of a new version of the patch set indicate that there is an active user community out there. Some members of the community are starting to wonder why the realtime preemption patches have not been merged, and when (if ever) that might change. The biggest reason is that Ingo has not yet requested that the patches be included - though many small pieces and fixes from the realtime patch set have found their way into the mainline. If and when Ingo does push for inclusion, however, there will be some opposition. 

To some developers, the realtime patch seems like a set of questionable and widespread changes aimed at the needs of a very small user community. Changing spinlocks into mutexes and moving interrupt handlers into threads are fundamental changes to how the kernel does things with the potential for the creation of subtle bugs and performance problems. Reworking things and adding complexity at that level is not a task that should be undertaken without a strong need - and many developers do not see a sufficiently strong need. 

There are some concerns about the performance impact of these changes. Acquiring an uncontended spinlock is a very fast operation; the rt_mutex type, with its wait queues and priority inheritance mechanisms, is bound to be slower. There is some [anecdotal evidence](<https://lwn.net/Articles/138184/>) that there is a performance hit to realtime preemption, but little in the way of real benchmarking has been done. In any case, the performance penalty should only affect users who have actually enabled the realtime preemption mode. 

Finally, not everybody is convinced that the realtime preemption approach can solve the real problem: providing an ironclad guarantee that a realtime process will be scheduled within a given maximum latency. Ingo believes that this guarantee can be made by eliminating all code within the kernel which can delay a reschedule; others feel that, to make a guarantee that can truly be trusted, the entire kernel must be audited and verified. They have a point: how strong a guarantee would you want before running realtime Linux in your car's braking system? 

Those who want true realtime guarantees, along with developers who simply do not want to clutter the kernel with realtime mechanisms, argue that a different approach should be taken. The most commonly suggested alternative is [RTAI-Fusion](<http://www.rtai.org/modules.php?name=Content&pa=showpage&pid=1>), which works (at its core) by interposing a "nanokernel" between Linux and the bare hardware. The nanokernel guarantees latency by taking the lowest-level scheduling decisions out of the Linux kernel's hands; it is kept small and easy to verify. Another project taking a similar approach is [Iguana](<http://www.disy.cse.unsw.edu.au/Software/Iguana/>), which is based on the [L4 microkernel](<http://www.l4hq.org/>). 

Since the realtime preemption patch is not being proposed for merging at this time, no decisions are likely to result from the current, lengthy discussion. If Ingo has [his way](<https://lwn.net/Articles/138188/>), there may never be one big decision; instead, pieces of the patch will be merged if and when it makes sense. 

So i'm afraid nothing radical will happen anywhere. Maybe we can have one final flamewar-party in the end when the .config options are about to be added, just for nostalgia, ok? 

There may be some interesting realtime-related sessions at next month's Kernel Summit in Ottawa, however. Meanwhile, should anybody wish to plow through the entire thread on linux-kernel, here is [the starting point](<https://lwn.net/Articles/138190/>).
