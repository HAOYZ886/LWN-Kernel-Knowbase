---
title: The next steps for userfaultfd()
url: https://lwn.net/Articles/718198/
date: "March 29, 2017"
category: userfaultfd
author: "By Jonathan Corbet March 29, 2017 LSFMM 2017"
---

By **Jonathan Corbet**  
March 29, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

The [`userfaultfd()`](<https://lwn.net/Articles/636226/>) system call allows user space to intervene in the handling of page faults. As Andrea Arcangeli and Mike Rapoport described in a 2017 Linux Storage, Filesystem, and Memory-Management Summit session dedicated to the subject, `userfaultfd()` was originally created to help with the live migration of virtual machines between physical hosts. It allows pages to be copied to the new host on demand, after the machine itself has been moved, leading to faster, more predictable migrations. Work on `userfaultfd()` is not finished, though; there are a number of other features that developers would like to add. 

In the 4.11 kernel, Arcangeli said, `userfaultfd()` can handle faults for missing pages, including anonymous, hugetlbfs, and shared-memory pages. There is also handling for a number of "non-cooperative events" [![\[Andrea
Arcangeli\]](https://static.lwn.net/images/conf/2017/lsfmm/AndreaArcangeli-sm.jpg)](<https://lwn.net/Articles/718201/>) (where the fault handler is unknown to the process whose faults are being managed) including mapping, unmapping, `fork()`, and more. At this point, though, only faults for not-present pages are managed; there would be value in dealing with other types of faults as well. 

In particular, Arcangeli is looking at write-protect faults, where the page is present in memory but is not accessible for writing. There are a number of use cases for this feature, many based on the idea that it allows the efficient removal of a range of memory from a region. That can be done with `munmap()` as well, but that results in split virtual memory area (VMA) structures and thus hurts performance. 

One potential use is efficient live snapshotting of running processes. The process could create a thread that would write the relevant memory to the snapshot. Memory that has been so written would then be write protected, generating faults when the main process tries to write there. Those faults can be used to copy the modified pages (and only those) to the snapshot. This feature could also be used to throttle copy-on-write faults, which are blamed for latency issues in some applications ([Redis, for example](<https://redis.io/topics/latency>)). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Another possible use case is getting rid of the write bits in language runtime engines. Getting faults on writes would allow the runtime to efficiently track which pages have been written to. Beyond that, it could help improve the robustness of shared-memory applications by catching writes to file holes. It could be used to notice when a malicious guest is trying to circumvent the balloon driver and use more memory than it has been allocated, implement distributed shared memory, or implement the long-desired [volatile ranges](<https://lwn.net/Articles/602650/>) functionality. 

At the moment, he has handling of write-protect faults working but it reports _all_ faults, not just those in the regions requested by the [![\[Mike
Rapoport\]](https://static.lwn.net/images/conf/2017/lsfmm/MikeRapoport-sm.jpg)](<https://lwn.net/Articles/718202/>) monitoring process. That, of course, means the monitor gets a lot of spurious events that must be filtered out. 

Rapoport talked briefly about the non-cooperative `userfaultfd()` mode, which was merged for the 4.11 kernel. It has been added mainly for the container case; it allows, for example, the efficient checkpointing of containers. Recent work has added events for `fork()`, `mremap()`, and `munmap()`, but there are still some holes, including the `fallocate()` `PUNCH_HOLE` command and `madvise(MADV_FREE)`. 

The handling of events is currently asynchronous, but, for this case, Rapoport said, there would be value in supporting synchronous events as well. There are also problems with pages shared between multiple processes resulting in the creation of multiple copies. Fixing that would require an operation to inject a single page into multiple address spaces at once. 

Perhaps the trickiest remaining problem, though, is using `userfaultfd()` on processes that are, themselves, using `userfaultfd()`. Fixing that will require adding a mechanism that allows the chaining of events. The first process (the checkpoint/restart mechanism, for example) would get all events, including a notification when the monitored process starts using `userfaultfd()` too. After that, events could be handled directly or passed down to the next level. There are a number of unanswered questions around nested use of `userfaultfd()`, though, so a complete solution is probably some time away.
