---
title: "The Big Kernel Semaphore?"
url: https://lwn.net/Articles/102253/
date: "September 15, 2004"
category: Big kernel lock; Latency
author: 
---

Much of the latency reduction work spearheaded by Ingo Molnar is reaching a state of completion; a lengthy set of patches has been posted which breaks up long lock hold times and adds "voluntary preemption" points at strategic places. With these patches in place, most of the worst latency problems in the 2.6 kernel have been addressed, even when kernel preemption is not enabled. That is good news for multimedia users and others who feel that their needs have been passed over in the 2.5/2.6 development period. 

One issue remains, however: there are some old parts of the kernel which still rely on the Big Kernel Lock (BKL) for mutual exclusion. Code which uses the BKL is not performance critical itself (all such uses have been fixed for a while). But the BKL is a lock, and code which holds the BKL will not be preempted. That can mean long latencies if a code path holds the BKL for a long time - and there are a few such paths. 

Interest in eradicating use of the BKL has waned in the last year or two, for a few reasons. Any code whose performance was seriously impacted by the BKL has been fixed. And, perhaps more to the point, much of the remaining code is ancient, crufty, and brittle. Finally, as Alan Cox (who holds the dubious fame of having created the BKL) [points out](<https://lwn.net/Articles/102254/>), the BKL is not a traditional lock: 

The BKL turns on old style unix non-pre-emptive sematics between all code that is within lock_kernel sections, that is it. That also makes it hard to clean up because lock_kernel is delimiting code properties (its essentially almost a function attribute) and spin_lock/down/up and friends are real locks and lock data. 

Fixing the remaining code is not an exercise for the timid. In most cases, the prudent course has been to simply leave things alone. The latency problem may just force this issue, however; by increasing latency, BKL-protected code is harming the higher-performance parts of the kernel. 

The BKL has one very interesting property which distinguishes it from an ordinary spinlock: code holding the BKL can call `schedule()` at any time. When that happens, the kernel releases the lock until the scheduling thread is returned to the processor. If code holding the lock can schedule, it ought to be preemptible as well - at least under some circumstances. 

Ingo Molnar has decided to mitigate the BKL problem by turning it into the Big Kernel Semaphore. As seen in [his patch](<https://lwn.net/Articles/102216/>), the BKS is a special sort of semaphore; it is recursive (as is the BKL), and it is released when the thread holding it voluntarily schedules. The key difference from the BKL, however, is that a process holding the BKS can be preempted - but the semaphore is not released in that case. So code which uses `lock_kernel()` is still protected against other such code, just like it is now. But that code can be preempted (as long as it does not take any spinlocks). That change should be sufficient to address the latency problems caused by long BKL hold times. 

Whether this patch will be accepted remains to be seen. Linus [doesn't like it](<https://lwn.net/Articles/102257/>), but Ingo [has reasonable responses](<https://lwn.net/Articles/102258/>) to his objections. Including Ingo's patch would mitigate the current problems caused by the BKL, which may have an undesirable effect: once again, there will be little motivation to truly fix users of the BKL. Some developers may prefer to simply bite the bullet and eliminate those final BKL users for real.
