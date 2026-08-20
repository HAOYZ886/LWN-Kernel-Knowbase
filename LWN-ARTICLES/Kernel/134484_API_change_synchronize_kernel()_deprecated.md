---
title: "API change: synchronize_kernel() deprecated"
url: https://lwn.net/Articles/134484/
date: "May 3, 2005"
category: "Read-copy-update; synchronize kernel"
author: 
---

The read-copy-update mechanism works with the fundamental assumption that, if no pointer to an RCU-protected data structure exists, there will be no references to that structure after every processor on the system has scheduled at least once. This assumption works because the rules require that accesses to RCU-protected data structures be atomic; scheduling while holding such a reference is not legal. When RCU was added to the kernel, it brought with it a function called `synchronize_kernel()` which would wait for every processor to schedule. Since it seemed that this capability could be useful outside of RCU itself, `synchronize_kernel()` was exported to the world. 

A quick grep of the 2.6.12-rc kernel shows a fair number of `synchronize_kernel()` calls. The module loader uses it to let things calm down when an attempted load fails. The AT keyboard driver calls it at disconnect time to ensure that no processor is still trying to work with the device. The kernel profiling code uses `synchronize_kernel()` to ensure that all processors notice the unregistration of its timer hook. And so on. 

The external uses of `synchronize_kernel()` have reached a point where they are putting extra demands on the RCU code. RCU, after all, does not _really_ have to wait until every processor has scheduled; the important constraint, instead, is that every processor running within `rcu_read_lock()` exits from the critical section. This distinction has become more important as the kernel developers have sought ways to make RCU more compatible with the low-latency work. 

So, as of 2.6.12-rc4, `synchronize_kernel()` will be officially deprecated. Its replacements will be `synchronize_sched()`, which retains the current "wait for all processors to schedule" semantics, and `synchronize_rcu()`, which is only guaranteed to wait until any processors executing within `rcu_read_lock()` critical sections have exited those sections. Most external users probably need to be switched over to `synchronize_sched()`. The comments suggest that a `synchronize_irq()` variant is also envisioned, but it has not been added as of this writing. 

One other significant change: unlike `synchronize_kernel()`, the two replacements are exported GPL-only.
