---
title: Recent RCU changes
url: https://lwn.net/Articles/894379/
date: "May 10, 2022"
category: "Read-copy-update"
author: "By Jake Edge May 10, 2022 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 10, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

In a combined filesystem and memory-management session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM), Paul McKenney gave an update on the changes to the read-copy-update (RCU) subsystem that had been made over the last several years. He started with a quick overview of what RCU is and why it exists at all. He did not go into any real depth, though, since many of the topics could take a 90-minute session of their own, he said, but he did provide some descriptions of the work that has gone into RCU recently. 

RCU is still under active development, McKenney said, which came as a big surprise to an academic person he was talking to at a conference a few years back. He did not have the heart to tell him that locking and atomic operations were also under active development, he said to laughter. "But here we are." 

#### RCU review

[ ![\[Paul McKenney\]](https://static.lwn.net/images/2022/lsfmm-mckenney-sm.png) ](<https://lwn.net/Articles/894523/>)

The overall problem is that global agreement in an operating system like Linux is expensive. While it would have shocked his 50-years-ago self, it turns out that the speed of light is too slow and atoms are too big; those things are causing massive problems in concurrent software these days. The way that is handled by RCU is to synchronize both in time and in space, he said. The core RCU API is split into two sets of temporal and spatial calls. RCU is a way to allow readers of a data structure to continue working with it while an updater makes a change; RCU is most often used with linked lists of various sorts. 

He showed a slide with four quadrants that described how RCU works on the temporal side. The basic idea is that readers call `rcu_read_lock()` before they read the data and `rcu_read_unlock()` after they are done. That allows an updater to remove the locked data, so long as it calls `synchronize_rcu()` (or receives the callback it set up with `call_rcu()`) before it actually frees it. The old data may still be in use by a reader, but RCU guarantees that all readers have unlocked it before it will return from `synchronize_rcu()`. The four quadrants show the possible orderings of a reader's lock/unlock and an updater's removal of the data, `synchronize_rcu()`, and the freeing of the old data. The one that could lead to serious misbehavior, where the removal is done after the lock and the freeing of the old memory is done before the unlock, is prevented by RCU. If that actually happens, it is a bug in RCU, he said. 

He also looked at the global agreement cost, contrasting the use of a reader-writer lock with that of RCU. When an updater wants to change the data using a reader-writer lock, there is a period of time before that lock has propagated to each of the readers, then the update can be done, but there is another lag waiting for that update to reach all of the readers. During all of that time, the readers are stalled waiting to be able to read again. Using RCU, that same time period is no longer wasted. Readers can continue, possibly with an outdated version of the data, while updaters just need to wait until the end of the grace period to ensure that all subsequent readers will see the new data. 

He showed some graphs to explain why the complexity of RCU is tolerated. RCU performs better than a reader-writer lock and it scales a _lot_ better as the number of threads goes up. In addition, the shorter the critical section and the more CPUs there are, the better RCU looks, he said. RCU can also prevent some deadlocks. 

More information about RCU can be found in the LWN kernel index [entry for RCU](<https://lwn.net/Kernel/Index/#Read-copy-update>). A good starting point is "[The RCU API, 2019 edition](<https://lwn.net/Articles/777036/>)". 

#### Changes

That ended his "full-speed review of RCU". He put up a list of eight changes to RCU that he wanted to talk about; he would guess at which of those were the most important to the audience and dig into them a bit deeper. He began with flavor consolidation. 

One day back in 2018, he got an email from Linus Torvalds, with a CC to security@kernel.org, that described an exploit in a use case of RCU. The problem was that the readers were using the `*_sched()` flavor for locking, but the updater was using `synchronize_rcu()` (instead of `synchronize_sched()`). In Linux 4.19 and earlier, the flavors needed to be matched up. The resulting bug was somehow used to cause a use-after-free, leading to the exploit. 

Torvalds asked if there was a way to deal with this problem; the way to do so is to make `synchronize_rcu()` work for all of the flavors. McKenney said that it took "about a year of my life to make it happen", but it solves the problem except for one little thing: if you need to backport something to 4.19 or earlier. Amir Goldstein asked: "does that mean we have a booby trap now?" McKenney agreed that it was, but that there is a "'get out of jail free' card" with `synchronize_rcu_mult()`; it can be called in those earlier kernels and gets passed the various flavors of `call_rcu()` that are being used. It will chain calls to each of those and wait for all of them before returning, which emulates the more recent version of `synchronize_rcu()` at the cost of some additional latency. 

In 5.4, Joel Fernandez added lockdep support to `list_for_each_entry_rcu()` (and the `hlist_*` variant). Those calls can take an optional lockdep expression, which removed the need for more variants of those calls in the API. 

Uladzislau Rezki added a single-argument version of `kvfree_rcu()` (`kfree_rcu()` is another name for it) in 5.9. Previously, `kvfree_rcu()` required two arguments: the object to be freed and the name of a field within the object containing an `rcu_head` structure. Now, adding the `rcu_head` to the object is optional; the name of the field is not passed in that case. If the object structures are small, the `rcu_head` may add more overhead than is desired. The two-argument version is still supported and, as always, never sleeps, but the new version can sleep if the system is out of memory. It is a tradeoff: you can use smaller structures, but it can sleep, he said. 

There are new variants of RCU that are for specialized use cases. He did not go into much detail, but wanted people to be aware of RCU Tasks Rude and RCU Tasks Trace since they may appear in tracebacks. They are mostly used for tracing of trampolines and he suggested that those who think they should use them contact him or one of the other users before doing so. RCU Tasks has been around since 3.18, but Rude and Trace were added in 5.8 with the help of Neeraj Upadhyay. 

Polling for the end of the grace period, instead of calling `synchronize_rcu()` and waiting for it goes back to 3.14. Polling is done by getting a cookie, then eventually passing the cookie to `cond_synchronize_rcu()`. This method works, but cannot be used in contexts where sleeping is not allowed. In addition, getting the cookie does not imply that the grace period has actually started, which can be problematic in some use cases. In 5.12, some functions were added to the API, `start_poll_synchronize_rcu()` and `poll_state_synchronize_rcu()`, along with `*_srcu()` variants for sleepable RCU, in order to support those use cases. There are some caveats to be aware of in using them, however. 

A feature that is mostly of interest to the realtime and HPC communities is the run-time callback offloading (and deoffloading) support that was added by Frédéric Weisbecker in 5.12. Normally, the RCU callbacks are executed on the CPU where they were queued, but that can interfere with the workload running on the CPU. So there is a way to offload those callbacks to kernel threads (kthreads) and then assign those kthreads elsewhere. 

Traditionally, those assignments were done at boot time by choosing which CPUs would be used for the callbacks and that could not be changed without rebooting. Weisbecker added the infrastructure that allows changing those assignments at run time; a CPU gets marked as "possibly offloaded" at boot time, then it can be switched to offloaded and back to deoffloaded at any time. Currently there is an internal kernel function to do so, but McKenney thinks it will be wired up with a user-space interface at some point. 

Another feature is a "memory diet" for the sleepable RCU (SRCU) code. Previously, it would allocate an array based on `NR_CPUS`, which is the maximum number of CPUs that the kernel can handle. That number is sometimes set to 4096 by distributions even though the vast majority of the systems where they run will have far fewer CPUs. So, instead of allocating the array at build time, it now gets allocated at run time based on the number of CPUs actually present. That is due in 5.19. 

Another feature slated for 5.19 is realtime expedited grace periods contributed by Rezki. McKenney gave a brief history of the length of RCU CPU-stall timeouts. In the 1990s, Dynix/PTX used 1.5s; in the 2000s, Linux used 60s, which was somewhat disappointing to him. In the 2010s that dropped to 21s for Linux; now a patch has proposed 20ms. On Android systems, the expedited grace period CPU-stall timeout would be 20ms, while it would stay 21s on other systems. 

In order for that to work, some additional patches from Kalesh Singh are being added. Normally expedited grace periods are driven by workqueues and run with the `SCHED_OTHER` scheduling class, like normal user-space processes. The patches will add a new kind of expedited kthread in the `SCHED_FIFO` scheduling class, which is "strong medicine", he said. It is limited to systems with fewer than 32 CPUs, no realtime, and with priority boosting enabled. The test results were impressive, he said, with latencies reduced by three orders of magnitude, down to roughly 2ms. It is a kind of realtime system with the expedited grace period on the fast path; "If you told me that last year, I would have laughed in your face." 

#### Future

He said that he had turned 100 this year, or perhaps 40, but in base 10, of course, that's 64\. He expects to be around for a while, noting that his father and grandfathers worked until they were 90 or so, but "mother nature's retirement program" awaits us all, so it is good to be prepared. He put up a list of some things that might be worked on in the future, but pointed out that the things that he can't see coming complicate that picture. There need to be people with a good understanding of RCU to handle those when they arise. 

He looked back at the commits to RCU over a two-year period that ended in April 2017, so five years ago. That showed 46 contributors, most of whom contributed a single patch, while McKenney contributed the vast majority of patches (288 patches or 74%). Looking at the previous two years from April 2022 shows 79 contributors, with McKenney's percentage of the patches committed dropping to 63% (503 patches). One reason that the overall patch numbers have increased is that, since he started at Facebook, he has concentrated on adding more distributed testing for RCU. 

In general, the trend is going in the right direction. There are developers who have been doing significant work deep inside RCU recently, which is great. There is still a lot of work to do, however, he said. One thing that he has noted over the years is that once a developer shows they can work on RCU, some company pays them a lot of money to work on something else. That is good, because people with some RCU knowledge are spread around the community. More recently he has noticed developers sticking with RCU itself, which is even better. 

The knowledge and understanding of RCU needs to be better propagated throughout the community, he said. He has recommended two presentations that he did as good starting points for that ([here](<https://www.linuxfoundation.org/webinars/unraveling-rcu-usage-mysteries/>) and [here](<https://linuxfoundation.org/webinars/unraveling-rcu-usage-mysteries-additional-use-cases/>)) but more is needed. There is also a more general problem of how to choose the right synchronization tool for a given problem—RCU is not always the right choice—which is another area that needs better understanding and propagation within the kernel community.
