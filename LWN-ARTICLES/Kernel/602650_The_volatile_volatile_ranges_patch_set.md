---
title: The volatile volatile ranges patch set
url: https://lwn.net/Articles/602650/
date: "June 18, 2014"
category: Volatile ranges
author: "By Jonathan Corbet June 18, 2014"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
June 18, 2014

"Volatile ranges" is a name given to regions of user-space memory that can be reclaimed by the kernel when memory is tight. The classic use case is for a web browser's image cache; the browser would like to keep that information in memory to speed future page loads, but it can do without that data should the memory used for the cache be needed elsewhere. Implementations of the volatile range concept have experienced more than the usual amount of change; that rate of change may well continue into the future — if a developer can be found to continue the work. 

[Early versions](<https://lwn.net/Articles/468896/>) of the patch set were based on the `posix_fadvise()` system call. Some developers complained that it was more of an allocation-related concept, so the patch was [reworked](<https://lwn.net/Articles/500382/>) to use `fallocate()` instead. By 2013, the plan [had shifted](<https://lwn.net/Articles/548111/>) toward the addition of two new system calls named `fvrange()` and `mvrange()`. [Version 11](<https://lwn.net/Articles/590991/>), released in March 2014, moved to a single system call named `vrange()`. During all of these iterations, there have also been concerns about user-space semantics (what happens when a process tries to access a page that has been purged, in particular) and the best way to implement volatile ranges internally. So nothing has ever been merged into the mainline kernel. 

[Version 14](<https://lwn.net/Articles/596770/>), posted by John Stultz on April 29, changes the user-space API yet again. Volatile ranges have now shifted to the `madvise()` system call. In particular, a call to: 

```
madvise(address, length, MADV_VOLATILE);
```

Will mark the memory range of `length` bytes starting at `address` as being volatile. Once the memory range has been marked in this way, the kernel is free to reclaim the associated pages and discard their contents at any time. Should the application need access to the range in the future, it should mark it as being nonvolatile with: 

```
madvise(address, length, MADV_NONVOLATILE);
```

The return value is zero for success (the range is now nonvolatile and the previous contents remain intact), a negative number if some sort of error occurred, or one if the operation was successful but at least one of the pages has been purged. 

The use of `madvise()` had been considered in the past; it makes sense, given that the purpose is to advise the kernel about the importance of a particular range of memory. Previous volatile range implementations, though, had the property that marking a range nonvolatile could fail partway through. That meant that the interface had to be able to return two values: (1) how many pages had been successfully marked, and (2) whether any of them had been purged. This time around, John found a way to make the operation atomic, in that it either succeeds or fails as a whole. In the absence of a need for a second return value, the `madvise()` interface is adequate for the task. 

What happens if user space attempts to access a volatile page that has been purged by the kernel? This implementation will deliver a SIGBUS signal in that situation. A properly-equipped application can catch the signal and respond by obtaining the needed data from some other source; applications that are not prepared will litter the disk with unsightly core dumps instead. That may seem like an unfriendly response, but one can argue that an application should not be trying to directly access memory that, according to instructions it gave to the kernel, does not actually need to be kept around. 

Minchan Kim [does not like](<https://lwn.net/Articles/602662/>) this approach; he would prefer, instead, that the application simply receive a new, zero-filled page in this situation. He is, it turns out, thinking about a slightly different use case: code that reuses memory and wants to tell the kernel that the old contents need not be preserved. In this case, the reuse should be as low-overhead as possible; Minchan would prefer to have no need for either an `MADV_NONVOLATILE` call or a SIGBUS signal handler. John [suggested](<https://lwn.net/Articles/602663/>) that Minchan's own [`MADV_FREE` patch](<https://lwn.net/Articles/590991/>) was better suited to that use case, but Minchan [disagreed](<https://lwn.net/Articles/602664/>), noting that `MADV_FREE` is a one-time operation, while `MADV_VOLATILE` can "stick" to a range of memory through several purge-and-reuse cycles. John, however, worries that silently substituting zero-filled pages could lead to data corruption or other unpleasant surprises. 

Johannes Weiner, who joined the conversation in June, also [prefers](<https://lwn.net/Articles/602665/>) that purged pages be replaced by zero-filled pages on access. He asked if the patch set could be reworked on top of `MADV_FREE` (which, he thinks, has a better implementation internally) to provide a choice: applications could request either the new-zero-filled-page or the SIGBUS semantics. John [responded](<https://lwn.net/Articles/602691/>) that he might give it a try, someday: 

I'll see if I can look into it if I get some time. However, I suspect its more likely I'll just have to admit defeat on this one and let someone else champion the effort. Interest and reviews have seemingly dropped again here and with other work ramping up, I'm not sure if I'll be able to justify further work on this. 

John certainly cannot be faulted for a lack of effort; this patch set has been through fourteen revisions since 2011; it has also been the subject of sessions at the Kernel Summit and Linux Storage, Filesystem, and Memory Management Summit. It has seen extensive revisions in response to comments from several reviewers. But, somehow, this feature, which has real users waiting for it to show up in a mainline kernel, does not seem much closer to being merged than before. 

At the same time, it is hard to fault the reviewers. The volatile ranges concept adds new user-visible memory-management behavior with some subtle aspects. If the implementation and interface are not right, the pain will be felt by developers in both kernel and user space for a long time. Memory-management changes are notoriously hard to get into the kernel for a good reason; user-visible changes are even worse. This patch set crosses two areas where, past history shows, we have a hard time getting things right, so some caution is warranted. 

Still, one can't help but wonder if merging nothing at all yields the best kernel in the long run. Users will end up working with out-of-tree variants of this concept (Android's "ashmem" in particular) that the development community has even less control over. Unless somebody comes up with the time to continue trying to push this patch set forward, the mainline kernel may never acquire this feature, leaving users without a capability that they demonstrably have a need for.
