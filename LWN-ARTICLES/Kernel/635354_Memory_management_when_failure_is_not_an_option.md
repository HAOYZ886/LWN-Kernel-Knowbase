---
title: Memory management when failure is not an option
url: https://lwn.net/Articles/635354/
date: "March 4, 2015"
category: "Memory management-Page allocator"
author: "By Jonathan Corbet March 4, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
March 4, 2015

Last December, a discussion of system stalls related to low-memory situations led to the revelation that [small memory allocations never fail](<https://lwn.net/Articles/627419/>) in the kernel. Since then, the discussion on how to best handle low-memory situations has continued, focusing in particular on situations where the kernel cannot afford to let a memory allocation fail. That discussion has exposed some significant differences of opinion on how memory allocation should work in the kernel. 

#### Some introductory concepts

The kernel's memory-management subsystem is charged with ensuring that memory is available when it is needed by either the kernel or a user-space process. That job is easy when a lot of memory is free, but it gets harder once memory fills up — as it inevitably does. When memory gets tight and somebody is requesting more, the kernel has a couple of options: (1) free some memory currently in use elsewhere, or (2) deny (fail) the allocation request. 

The process of freeing (or "reclaiming") memory may involve writing the current contents of that memory to persistent storage. That, in turn, involves calling into the filesystem or block I/O code. But if any of those subsystems are, in fact, the source of the allocation request, calling back into them can lead to deadlocks and other unfortunate situations. For that reason (among others), allocation requests carry a set of flags describing the actions that can be performed in the handling of the request. The two flags of interest in this article are `GFP_NOFS` (calls back into filesystems are not allowed), and `GFP_NOIO` (no type of I/O can be started). The former inhibits attempts to write dirty pages back to files on disk; the latter can block activity like writing pages to swap. 

Obviously, the more constrained the memory-management subsystem is, the higher the chances of it being unable to satisfy an allocation request at all. Kernel developers have long been told that (almost) any allocation request can fail; as a result, the kernel is full of error-handling paths meant to deal with that eventuality. But it became clear recently that the memory-management code does not actually allow smaller requests to fail; it will, instead, loop indefinitely trying to free some memory. That behavior has been seen to lead occasionally to locked-up systems, despite the fact that the code involved is prepared to deal with allocation failures. The "too small to fail" behavior is controversial, but would prove hard to change at this point. 

There are, however, places in the kernel that are simply unprepared to deal with allocation failures, usually because the allocation happens deep within a complex series of operations that would be difficult to unwind. The `__GFP_NOFAIL` flag exists to explicitly state that failure is not an option for a given request, though its use is heavily discouraged. 

The following discussion, in the end, focuses on two related questions: (1) should the kernel really be treating small allocations as if they all had `__GFP_NOFAIL` set, and (2) should failure-proof allocations be supported at all, and, if so, how can that support be made more robust? 

#### No longer too small to fail

The discussion (re)started when Tetsuo Handa [noted](<https://lwn.net/Articles/635360/>) that memory allocation behavior had changed in the 3.19 kernel; in particular, small allocations with the `GFP_NOIO` or `GFP_NOFS` flags would fail under severe memory pressure. In previous kernels, such allocations would loop indefinitely if no memory was available. Among other things, this change can cause filesystem operations to fail on memory-stressed systems where they would have (eventually) succeeded before. 

The behavior change is the result of [this patch from Johannes Weiner](<http://git.kernel.org/linus/9879de7373fcfb46>) which was aimed at avoiding the memory-allocation deadlocks that started the December discussion. The intent was to avoid looping forever in an allocation attempt if it appeared that no progress was being made toward freeing some memory for that allocation, but, by accident, it also prevented looping entirely in the `GFP_NOIO` and `GFP_NOFS` cases. So those allocations can now fail; that is a significant change from how previous kernels worked. 

Johannes initially [wanted to keep the new behavior](<https://lwn.net/Articles/635361/>), saying that it ""makes more sense"". But the filesystem developers disagreed strongly. It seems that there are numerous places in the filesystem code that depend on allocations succeeding reliably, and that many of them are not marked with `__GFP_NOFAIL`. Ted Ts'o [threatened](<https://lwn.net/Articles/635362/>) to add a lot of `__GFP_NOFAIL` flags to allocation calls in the ext4 filesystem if the change were not reverted. The memory-management developers were thus faced with the need to pick the option they disliked least. 

In the end, the filesystem developers won out on this one; Johannes merged [a change](<http://git.kernel.org/linus/cc87317726f851531ae8422e0c2d3d6e2d7b1955>) into 4.0-rc2 restoring the looping behavior for those allocation types. This change is likely to end up in the 3.19 stable series as well. The original patch is a good argument for the approach of refusing "cleanup" patches late in the development cycle. It was merged for the 3.19-rc7 prepatch, meaning that there was almost no time for problems to be noticed before the final 3.19 release came out. 

The discussion was not limited to the unexpected effects of one late-arriving memory-management patch, though. The bigger problems of how to avoid deadlocks in low-memory situations and how to ensure that important tasks can proceed in those situations remain unsolved. 

#### The OOM killer

The out-of-memory (OOM) killer is implicated in a number of stall scenarios. In the original problem reported last December, the OOM killer would choose a victim that was blocked on a lock, but that lock was held by the process waiting (forever) for a memory allocation to proceed. As a result, the victim could not exit and, thus, could not free its memory. Since the OOM killer only goes after a single process at a time, everything would stop at that point. 

Johannes [suggested a change](<https://lwn.net/Articles/635364/>) to how the OOM killer works: if a targeted process failed to exit after five seconds, the OOM killer would give up and move on to another victim. The idea was not hugely popular, though. David Rientjes [pointed out](<https://lwn.net/Articles/635365/>) that there was no guarantee that the next victim would be any more appropriate than the one that came before. Dave Chinner [claimed](<https://lwn.net/Articles/635367/>) more broadly that efforts to tweak the OOM killer are misdirected: 

I really don't care about the OOM Killer corner cases - it's completely the wrong line of development to be spending time on and you aren't going to convince me otherwise. The OOM killer a crutch used to justify having a memory allocation subsystem that can't provide forward progress guarantee mechanisms to callers that need it. 

The end result is that OOM-killer timeouts will probably not find their way into the memory-management subsystem anytime soon. 

#### __GFP_NOFAIL and looping

From the point of view of the memory-management developers, many things would get easier if any allocation request could fail when the necessary resources are not available. That would mean getting rid of the implicit "small allocations never fail" rule, but, beyond that, it would also require getting rid of the explicit `__GFP_NOFAIL` call sites. Michal Hocko was perhaps the most outspoken in this regard, [saying](<https://lwn.net/Articles/635376/>) that `__GFP_NOFAIL` ""is deprecated and shouldn't be used"". He also suggested that existing `__GFP_NOFAIL` call sites should be reimplemented in a way that allows them to recover from allocation failures. 

Dave [took issue](<https://lwn.net/Articles/635378/>) with that idea, saying that failure-proof allocations are a hard requirement for the XFS filesystem. To rework XFS to be able to roll back dirty transactions in the face of an allocation failure would increase its complexity significantly, he said; the project would take a couple of years to reach a point where it could be put into production use. He summarized by saying ""I'm not about to spend a couple of years rewriting XFS just so the VM can get rid of a GFP_NOFAIL user"". Strangely enough, there were no other developers volunteering to take on that job either. 

Contemporary filesystems are complex beasts that have to meet a wide variety of demands. They incorporate complex transaction mechanisms that help them to maintain filesystem integrity in every situation possible. Implementing such a mechanism in a way that allows it to recover from a memory-allocation failure in the middle of a transaction, after resources have been committed, locks taken, etc., is not a simple task. Filesystem developers on Linux have not taken on that task because, in the end, there has not been a need to. Allocations that cannot be allowed to fail have proved sufficient in almost all situations. 

Once one accepts that some sort of failure-proof allocation mechanism is needed, though, the next question is: how should it be done? The `__GFP_NOFAIL` flag is one solution, but it turns out that quite a bit of code in the kernel does not make use of it. Instead, there are a number of places in the kernel that implement their own retry loops on top of a `kmalloc()` call without `__GFP_NOFAIL`. That is something that the memory-management developers don't like; those developers would rather not see `__GFP_NOFAIL` used at all, but they still prefer its use to retry loops implemented outside of the memory-management subsystem. Consider, for example, [this message from Johannes](<https://lwn.net/Articles/635381/>) saying that the XFS developers should replace a retry loop with a single `__GFP_NOFAIL` call. 

There are couple of reasons why such loops exist. One of those is that `__GFP_NOFAIL` was [explicitly deprecated](<http://git.kernel.org/linus/dab48dab37d2770824420d1e01730a107fade1aa>) in 2009; the patch (from Andrew Morton) said: 

__GFP_NOFAIL is a bad fiction. Allocations _can_ fail, and callers should detect and suitably handle this (and not by lamely moving the infinite loop up to the caller level either). 

After this change went in, it became harder to get code containing `__GFP_NOFAIL` past reviewers. Whether it is done lamely or not, a hand-coded infinite retry loop is easier to sneak into the kernel than an easily greppable `__GFP_NOFAIL` use. So that is what developers did. 

The memory-management developers dislike must-succeed allocations because they complicate the code and, as has occasionally been seen, create the possibility of deadlocks. If such allocations must be made, they would rather see the looping done in the memory-management code, where behavior can be tweaked and appropriate action taken (starting the OOM killer, for example) if it becomes clear that no progress is being made. In the real world, though, according to both [Ted](<https://lwn.net/Articles/635383/>) and [Dave](<https://lwn.net/Articles/635384/>), looping actually works pretty well. The XFS code has a "canary" that puts out a warning when the looping goes on for too long, but, Dave said: 

yet we *rarely* see the canary warnings we emit when we do too many allocation retries, the code has been that way for 13-odd years. Hence, despite your protestations that your way is *better*, we have code that is tried, tested and proven in rugged production environments. That's far more convincing evidence that the *code should not change* than your assertions that it is broken and needs to be fixed. 

One might take that as a statement that the XFS developers are currently uninterested in replacing their own loops with `__GFP_NOFAIL` invocations. But they actually have another reason to maintain a loop outside of the memory-management code: [they want to retain control](<https://lwn.net/Articles/635385/>) over how the filesystem should respond to low-memory conditions. It is, in their mind, a policy decision that the memory-management code lacks the information to handle. There are currently plans afoot to expose some of that policy to user space, allowing administrators to configure what the filesystem's low-memory response should be. 

#### Reservations

Still, there is no real disagreement over this idea: looping over a failing memory allocation is undesirable and best avoided whenever possible. Thus it may well be that the most useful part of the discussion came when the developers got around to the topic of avoiding allocation failures altogether. There are a few ways of working toward that goal. 

One of those is preallocation — allocating all of the needed memory resources before the code gets to a point where it can't back out of a transaction. Preallocation is used in many contexts in the kernel and works well, so it was natural for the memory-management developers to ask whether it can be used in this context. Dave [shot that idea down](<https://lwn.net/Articles/635387/>) fairly quickly: 

However, preallocation is dumb, complex, CPU and memory intensive and will have a *massive* impact on performance. Allocating 10-100 pages to a reserve which we will almost *never use* and then free them again *on every single transaction* is a lot of unnecessary additional fast path overhead. Hence a "preallocate for every context" reserve pool is not a viable solution. 

[Mempools](<https://lwn.net/Articles/22909/#mempool>) were also raised as a possibility. They are a form of preallocation that might avoid some of the overhead described above. But they are, [according to Dave](<https://lwn.net/Articles/635389/>), poorly suited to the problem at hand. Mempools deal with a single size of object, while a filesystem transaction needs a wide variety of objects; that implies that several mempools would be needed at various levels in the stack. There is also a mismatch between object lifetimes that make mempools difficult to use across multiple transactions. So mempools do not appear to be an option either. 

Dave's suggestion, instead, is to add the concept of "reservations" to the memory-management subsystem. Prior to entering a transaction, the filesystem code would inform the memory-management code that it will need guaranteed access to a certain amount of memory; calculating an approximate memory requirement is, apparently, not that hard. The memory-management code would then ensure that the requisite amount of memory would be available; subsequent allocation requests would dip into the reserve if need be. As long as the estimate for the size of the reserve is sufficient, there should be no problem with failing allocations during the transaction. 

Reservations may look a lot like preallocation, but there is a crucial difference. The memory-management code already maintains a "watermark," a level of free memory below which it is unwilling to go unless absolutely necessary. A reservation would simply raise that watermark, making a bit less memory available to the system as a whole. If a reservation would raise the watermark above the amount of memory that is currently free, the request would block until more memory could be reclaimed. In the simplest case, a reservation would be represented as an increased value in a single integer variable. 

There seems to be some general support for the addition of a reservation mechanism, but things get less clear once one looks at the details. Andrew Morton [suggested a scheme](<https://lwn.net/Articles/635390/>) where a process making a reservation would get a number of "tokens"; subsequent allocations done by that process would come from the reserve first. Dave [does not like that idea](<https://lwn.net/Articles/635391/>), saying that it fails to account for the fact that many objects allocated during a transaction will be freed (perhaps by others) shortly and, thus, should not come from the reservation. His view of the reservation, instead, is a range of memory that is not touched at all unless there is no alternative; even then, only allocations using the `GFP_RESERVE` flag would be able to get at that memory. The reservation, in his view, comes into play when the kernel would have otherwise put the OOM killer into action. 

Johannes, instead, [says](<https://lwn.net/Articles/635392/>) that this approach will not work. The problem is that ""we simply don't KNOW the exact point when we ran out of reclaimable memory"", so the memory-management subsystem cannot easily guarantee the sort of loose reservation that Dave has described. Dave disagreed with that assessment, it almost goes without saying. And that is about where the conversation wound down. 

Reservations are a promising idea for a solution to some of the kernel's memory-allocation challenges. But, at this point, it is just an idea; it has neither code nor a design consensus behind it. The discussion has slowed for the moment, but that is almost certain to be a temporary state of affairs. The annual [Linux Storage, Filesystem, and Memory Management Summit](<http://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit>) is less than one week away as of this writing. This subject is on [the agenda](<https://events.linuxfoundation.org/events/linux-storage-filesystem-and-mm-summit/program/schedule>), and LWN will be there to report on the discussion.
