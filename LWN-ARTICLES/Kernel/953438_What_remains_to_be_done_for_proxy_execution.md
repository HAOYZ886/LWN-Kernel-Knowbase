---
title: What remains to be done for proxy execution
url: https://lwn.net/Articles/953438/
date: "December 4, 2023"
category: "Scheduler-Proxy execution"
author: "By Jonathan Corbet December 4, 2023 LPC"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
December 4, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

The kernel's [deadline scheduling class](<https://lwn.net/Articles/743740/>) offers a solution to a number of realtime (or generally latency-sensitive) problems, but it is also resistant to the usual solutions for the [priority-inversion](<https://en.wikipedia.org/wiki/Priority_inversion>) problem. The development community has been pursuing proxy execution as a solution to a few scheduling challenges, including this one; the problem is difficult and progress has been slow. LWN last [looked at proxy execution](<https://lwn.net/Articles/934114/>) in June; at the [2023 Linux Plumbers Conference](<https://lpc.events/event/17/page/198-lpc-2023-overview>), John Stultz gave an overview of proxy execution, the current status of the work, and the remaining problems to solve. 

The current work is (at least partially) driven by the needs of the Android system, where it would be useful to run some tasks with realtime priorities. Android has run into a lot of priority-inversion problems while trying, though, and the classic solution for such problems (priority inheritance) doesn't work for deadline or non-realtime tasks in Linux. As a result, Android systems cannot throttle background task activity in any useful way without creating inconsistent behavior for the foreground tasks that the user actually cares about at the moment. 

[![\[John Stultz\]](https://static.lwn.net/images/conf/2023/lpc/JohnStultz-sm.png)](<https://lwn.net/Articles/953445/>) Priority inversion comes about when a lower-priority task holds a resource that is needed by a higher-priority task; if the lower-priority task is unable to obtain CPU time to run, it may hold that resource for a long time, preventing the high-priority task from running. Solutions usually involve having the waiting task lend its priority to the task holding the resource it needs, so that the holding task can run and release the resource. Deadline tasks in Linux, though, have no priority, so a different approach is needed. 

The core idea behind proxy execution is to track the "blocked-on" relationship between tasks waiting for a resource and the task owning that resource. In current kernels, a task that is waiting for a resource will be removed from the run queue; with proxy execution, instead, it would be left on the queue as if it were still runnable. Should the scheduler actually select the blocked task to run, it will follow the blocked-on link to run the holder of the resource instead. The wait task, in effect, donates some of the run time it would have been given toward the goal of freeing the resource it needs. 

That idea is simple, but the implementation turns out to be complex. The task holding a resource may, in turn, be blocked on a second resource, so the scheduler may have to follow a chain of blocked-on links, of arbitrary length, to get to a task that can actually be run. The blocked-on links can cross CPUs; before the scheduler can run the task pointed to by such a link, it will need to migrate the blocked task to the CPU where the holder is actually running. It might be that the resource holder at the end of a blocked-on chain is currently sleeping (unable to run) for some other reason, meaning no progress can be made until that task can run again. In this case, the originally blocked task is enqueued onto the sleeping task that holds the resource, so that both can be awakened together. There are other challenges as well. 

Even so, proxy execution shows some real promise. Stultz showed results from workloads that, on current kernels, experienced occasional large latencies as the result of priority inversion. The addition of proxy execution made those latencies go away. So proxy execution is "very attractive" for the Android use case, he said. 

There have been three versions of this work posted since it was [presented at the OSPM conference](<https://lwn.net/Articles/935180/>) in April. He has been reworking the patch series to make it bisectable; [version 6](<https://lwn.net/ml/linux-kernel/20231106193524.866104-1-jstultz@google.com/>) of the patch set was posted in early November. Among other things, that version has partially fixed a performance regression that was introduced in the previous round. 

There are a few outstanding problems, he said, as he approached the end of his prepared talk. The handling of the sleeping-owner problem described above is hard to get right; the intent is to enqueue the waiting (higher-priority) task onto the owning task, so that they can wake together, but that is complex. Stultz worried that implementing that solution could involve reinventing the kernel's run queues. If a task has been migrated to lend its execution time to a resource holder, it should be migrated back once the blockage has been resolved, but that is slow and involves locking challenges. The performance regression introduced in version 5 still has not been fully resolved. There is also the little problem of adding some significant complexity to the scheduler, which is already a "terribly subtle" and difficult kernel subsystem. 

Stultz concluded by saying that he is still working to find the best way to break up the patch series. He wondered whether Android needed to ship a kernel with proxy execution before it would be seriously considered for the mainline. That would show the value of the work, but it would also increase the divergence between the Android and mainline kernels, which is something that the Android project would like to avoid. Once a feature has been added to Android, he said, it can be quite difficult to remove, since vendors quickly come to depend on it. He closed with a request for more reviews of the design and correctness of the proxy-execution patches. 

The ensuing discussion was mostly focused on reader/writer locks (rwlocks), which are not currently handled by the proxy-execution patches. Thomas Gleixner said that priority inheritance has, in the past, proved to not be an option for rwlocks; it ends up being far too complex. Boosting the holder of a write lock (in response to a higher-priority reader needing the lock) is easy, since there can only be one writer, but there can be multiple readers, complicating the picture. Having one blocked writer trying to boost a whole set of readers doesn't scale, he said, and "gets messy". 

Gleixner said that, in the realtime tree, they gave up and settled for just boosting writers; that improves the situation, but leaving readers unboosted can lead to writer starvation. That said, perhaps the problem is not as big of a concern now. Fifteen years ago, there were a lot of highly contended rwlocks in the kernel, but many of the users of rwlocks, especially in the networking subsystem, have switched to using read-copy-update instead. The one big holdout is the [`mmap_lock`](<https://lwn.net/Articles/893906/>), but code needing that lock is already running in a slow path and can afford to wait a bit. He has not seen any problems with writer starvation recently. So the real solution for rwlocks, he concluded, is to simply stop using them. 

At that point, the session ran out of time. For the curious, the [video](<https://www.youtube.com/live/Ow08lAlktkQ?si=zqfMYRg3aVNWyAyb&t=134m9s>) and [slides](<https://lpc.events/event/17/contributions/1482/attachments/1147/2382/LPC23%20-%20ProxyExecution.pdf>) from the talk are available. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event.]
