---
title: Ideas for rationalizing GFP flags
url: https://lwn.net/Articles/684437/
date: "April 20, 2016"
category: "Memory management-GFP flags"
author: "By Jonathan Corbet April 20, 2016 LSFMM 2016"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 20, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

The kernel's memory-allocation functions normally take as an argument a set of flags describing how the allocation is to be performed. These "GFP flags" (for "get free page") control both the placement of the allocated memory and the techniques the kernel can use to make memory available if need be. For some time, developers have been saying that these flags need to be rethought; in two separate sessions at the [2016 Linux Storage, Filesystem, and Memory-Management Summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit/>), Michal Hocko explored ways of doing that. 

#### GFP_REPEAT

The first session, in the memory-management track, started with a discussion of the `GFP_REPEAT` flag which, as its name would suggest, is meant to tell the allocator to retry an attempt should it fail the first time. This flag, Michal said, has never been useful. It is generally used for order-0 (single-page) allocations, but those allocations [are not allowed to fail](<https://lwn.net/Articles/627419/>) and, thus, will retry indefinitely anyway. For larger requests, he said, it "pretends to try harder," but does not actually do anything beneficial. Michal would like to clean this flag up and create a better-defined set of semantics for it. 

The kernel does have the opposite flag in the form of `GFP_NORETRY`, but that one, he said, is not useful for anything outside of order-0 allocations. What he would like to see instead is something he called `GFP_BESTEFFORT`; it would try hard to satisfy the request, but would not try indefinitely. So it could retry a failed request, and even invoke the out-of-memory killer but, should that prove fruitless, it would give up. This flag would be meant to work for all sizes of requests. 

He is trying to move things in that direction, starting with the removal of `GFP_REPEAT` from order-0 allocation requests around the kernel. The next step would be to start placing the new flag in the places where it makes sense. As an example, he mentioned transparent huge pages and the [![\[Michal Hocko\]](https://static.lwn.net/images/conf/2016/lsfmm/MichalHocko-sm.jpg)](<https://lwn.net/Articles/684487/>) hugetlbfs filesystem. Both need to allocate huge pages but, while an allocation failure for a transparent huge page is just a missed optimization opportunity, a failure in hugetlbfs is a hard failure that will be passed back to user space. It clearly makes sense to try harder for hugetlbfs allocations. 

Johannes Weiner asked whether it would be a good idea to provide best-effort semantics by default while explicitly annotating the exceptions where it is not wanted. The existing `GFP_NORETRY` flag could be used for that purpose. Michal said that doing so would cause performance regressions, leading Andrew Morton to question whether "taking longer but succeeding" constitutes a regression. The point is that some callers do have reasonable fallback paths for failed allocations and would rather see the failures happen quickly if they are going to. Andrew asked how often that sort of failure happens, but nobody appeared to have any sort of answer to that question. It will be, in any case, highly workload-dependent. 

Johannes persisted, saying that it can be difficult to know where the memory allocator should be told to try harder, but it is usually easy to see the places where failure can be handled easily. There was also a suggestion to make the flags more fine-grained; rather than use a vague "best effort" flag, have flags to specify that retries should not be done, or that the out-of-memory killer should not be invoked. Mel Gorman noted that he has already [done some work in that direction](<https://lwn.net/Articles/658081/>), adding flags to control how reclaim should be performed. 

That led to a wandering discussion on whether the flags should be positive ("perform direct reclaim") or negative ("no direct reclaim"). Positive flags are more descriptive, but they are a bit more awkward to use since call sites will have to mask them out of combined mask sets like `GFP_KERNEL`. There are also concerns that there aren't many flag bits available for fine-grained control. 

The session ended with Michal asking if the group could at least come to a consensus that his work cleaning up `GFP_REPEAT` made sense. There seemed to be no objection there, so that work can be expected to continue. 

#### GFP_NOFS

Later that day, the entire LSFMM group was present while Michal talked about a different GFP flag: `GFP_NOFS`. This flag instructs the memory allocator to avoid actions that involve calling into filesystem code — writing out dirty pages to files, for example. It exists for use by filesystem code for a number of reasons, the most straightforward of which is the avoidance of deadlocks. If a filesystem acquires locks then discovers that it must allocate memory, it doesn't want the allocator coming back and trying to obtain the same locks. But there is more to it than that; `GFP_NOFS` reflects a number of "indirect dependencies" within the filesystems. Also, XFS uses it for all page-cache allocations, regardless of deadlock concerns, to avoid calling so deeply into filesystem code that the kernel stack overflows. 

There are, Michal said, too many uses of `GFP_NOFS` in the kernel tree; they needlessly constrain the memory allocator's behavior, making memory harder to obtain than it should be. So he would like to clean them up, but, he acknowledged, that will not be easy. The reason for any given use of `GFP_NOFS` is often far from clear — if there is one at all. 

His suggestion is to get rid of direct use of that flag entirely; instead, setting a new task flag would indicate that that current task could not call back into filesystem code. XFS has a similar mechanism internally now; it could be pulled up and used in the memory-management layer. A call to a function like `nofs_store()` would set the flag; all subsequent memory allocations would implicitly have `GFP_NOFS` set until the flag was cleared. 

There are a number of reasons for preferring this mechanism. Each call to `nofs_store()` would be expected to include documentation describing why it's needed. It allows the "no filesystem calls" state to follow the task's execution into places — security modules, for example — that have no knowledge of that state. Chris Mason noted that it would save filesystem developers from sysfs, which brings surprises of its own. Ted Ts'o added that there are a number of places where code called from ext4 should be using `GFP_NOFS` for its allocations, but that doesn't happen because it would simply be too much work to push the GFP flags through the intervening layers. Thus far, he has been crossing his fingers and hoping that nothing goes wrong; this mechanism would be more robust. 

Michal asked the filesystem developers in the room how much work it would be to get rid of the `GFP_NOFS` call sites. Chris said that the default in Btrfs has been to use it everywhere; a bunch of those sites have since been fixed, but quite a few remain. He would be happy to switch to the new API, he said. Ted agreed, as long as the transition would be gradual and `GFP_NOFS` would not disappear in a flag day, as it were. The end result, he said, would be nice. 

There was some talk of refining the mechanism to specify the specific filesystem that should be avoided, allowing the memory allocator to call into other filesystems. The consensus seemed to be that this idea would be tricky to implement; the possibility of stack overruns was also raised. Michal will go ahead and put together an API proposal for review. He hopes it will succeed: the fewer `GFP_NOFS` sites there are, the better the memory allocator's behavior will be.
