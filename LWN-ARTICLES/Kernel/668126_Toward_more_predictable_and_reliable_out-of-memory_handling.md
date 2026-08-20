---
title: "Toward more predictable and reliable out-of-memory handling"
url: https://lwn.net/Articles/668126/
date: "December 16, 2015"
category: "Memory management-Out-of-memory handling; OOM killer"
author: "By Jonathan Corbet December 16, 2015"
---

By **Jonathan Corbet**  
December 16, 2015

The kernel's out-of-memory (OOM) behavior has been a topic of discussion almost since the inception of the linux-kernel mailing list. Opinions abound on how the kernel should account for memory, whether it should allow memory to be overcommitted, what it means to be out of memory, and what should be done when that situation comes about. There seems to be general agreement on only one thing: OOM situations are bad, and the kernel's handling of OOM situations is even worse. Over the years, numerous developers have tried to improve the situation; the latest attempt can be seen in two patch sets from Michal Hocko. 

#### OOM detection

The [first patch set](<https://lwn.net/Articles/667939/>) tries to answer a question that one might think would be obvious: how do we know when the system is out of memory? The problem is that a running system is highly dynamic. The lack of a free page to allocate at the moment does not mean that such pages could not be created; given the high cost of invoking the OOM killer, it is best not to declare an OOM situation if the kernel might be able to scrounge some memory from somewhere. Current kernels, though, are a bit unpredictable regarding when they give up and, in some cases, might wait too long. 

If there are no pages to satisfy an allocation request, the kernel will perform direct reclaim to try to free some memory. In some cases, direct reclaim will be successful; that happens, for example, if it finds clean pages that can be immediately repurposed. In other cases, though, reclaiming pages requires writing them back to backing store; those pages will not be available for what is, from a computer's perspective, a long time. Still, they should become available eventually, so the kernel is justifiably reluctant to declare an OOM situation for as long as reclaimable pages exist. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The problem is that there are no real bounds on how long it might take for "reclaimable" pages to actually be reclaimed, for a number of reasons. Additionally, the allocator can conceivably find itself endlessly retrying if a single page is reclaimed, even if that page cannot be used for the current allocation request. As a result, the kernel can find itself hung up in allocation attempts that do not succeed, but which do not push the system into OOM handling. 

Michal's patch defines a new heuristic for deciding when the system is truly out of memory. When an allocation attempt initially fails, the logic is similar to what is done in current kernels: a retry will be attempted (after an I/O wait) if there is a memory zone in the system where the sum of free and reclaimable pages is at least as large as the allocation request. If the retries continue to fail, though, a couple of changes come into play. 

The first of those is that there is an upper bound of sixteen retries; after that, the kernel gives up and goes into OOM-handling mode. That may bring about an OOM situation sooner than current kernels (which can loop indefinitely) will, but, as Michal [put it](<https://lwn.net/Articles/668133/>): ""OOM killer would be more appropriate than looping without any progress for unbounded amount of time"". Beyond that, the kernel's count of the number of reclaimable pages is discounted more heavily after each unsuccessful retry; after eight retries, that number will be cut in half. That makes it increasingly unlikely that the estimate of reclaimable pages will motivate the kernel to keep retrying. 

The result of these changes is that the kernel will go into OOM handling in a more predictable manner when memory gets tight. Users will still curse the results, but the system as a whole should more reliably survive OOM situations. 

#### The OOM reaper

At least, that should be the case if the OOM killer is actually able to free pages when the kernel invokes it. As [has been seen](<https://lwn.net/Articles/627419/>) in recent years, it is not that hard to create a situation where the OOM killer is unable to make any progress, usually because the targeted process is blocked on a lock and the OOM situation itself prevents that lock from being released. If an OOM-killed process cannot run, it cannot exit and, thus, it cannot free its memory; as a result, the entire OOM-killing mechanism fails. 

The observation (credited to Mel Gorman and Oleg Nesterov) at the core of Michal's [OOM reaper patch set](<https://lwn.net/Articles/666024/>) is that it is not necessary to wait for the targeted process to die before stripping it of much of its memory. That process has received a SIGKILL signal, meaning it will not run again in user mode. That, in turn, means that it will no longer access any of its anonymous pages. Those pages can be reclaimed immediately without changing the end result. 

The OOM reaper is implemented as a separate thread; this is done because the reaper must be able to run when it is called upon to do its work. Other kernel execution mechanisms, such as workqueues, might themselves be blocked by the OOM situation, so they cannot be counted upon. If this patch is merged, the `oom_reaper` thread will sit unused on the majority of Linux systems out there, but it will be certain to be available on the systems where it is needed. 

The reaper is not without its rough edges. It must still take the `mmap_sem` lock to free the pages, meaning that it could be blocked if `mmap_sem` is held elsewhere. Still, Michal says that the probability of trouble ""is reduced considerably"" compared to current kernels. One other potential problem is that, if the targeted process is dumping core at the time it is killed, removing its pages may corrupt the dump. This tradeoff is worthwhile, though, Michal says, since keeping the system running is more important in such situations. 

Memory-management patches are notoriously difficult to get merged into the kernel. With regard to the OOM detection patch, Michal said the work ""has been sitting and waiting for the fundamental objections for quite some time and there were none"". He would like to see it merged in 4.6 or thereafter. Objections to the OOM reaper have also been hard to find, but there has been no talk yet as to when that patch might head for the mainline. Once these patches get there, the OOM-handling subsystem may work a little better, but it seems unlikely that users will appreciate it any more than they do now.
