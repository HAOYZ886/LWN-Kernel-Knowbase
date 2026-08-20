---
title: "The \"too small to fail\" memory-allocation rule"
url: https://lwn.net/Articles/627419/
date: "December 23, 2014"
category: "Memory management-Page allocator"
author: "By Jonathan Corbet December 23, 2014"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
December 23, 2014

Kernel developers have long been told that, with few exceptions, attempts to allocate memory can fail if the system does not have sufficient resources. As a result, in well-written code, every call to a function like `kmalloc()`, `vmalloc()`, or `__get_free_pages()` is accompanied by carefully thought-out error-handling code. It turns out, though, the behavior actually implemented in the memory-management subsystem is a bit different from what is written in the brochure. That difference can lead to unfortunate run-time behavior, but the fix might just be worse. 

A discussion on the topic began when Tetsuo Handa posted [a question](<https://lwn.net/Articles/627420/>) on how to handle a particular problem that had come up. The sequence of events was something like this: 

  1. A process that is currently using relatively little memory invokes an XFS filesystem operation that, in turn, needs to perform an allocation to proceed. 

  2. The memory management subsystem tries to satisfy the allocation, but finds that there is no memory available. It responds by first trying direct reclaim (forcing pages out of memory to free them), then, if that doesn't produce the needed free memory, it falls back to the out-of-memory (OOM) killer. 

  3. The OOM killer picks its victim and attempts to kill it. 

  4. To be able to exit, the victim must perform some operations on the same XFS filesystem. That involves acquiring locks that, as it happens, the process attempting to perform the problematic memory allocation is currently holding. Everything comes to a halt. 

In other words, the allocating process cannot proceed because it is waiting for its allocation call to return. That call cannot return until memory is freed, which requires the victim process to exit. The OOM killer will also wait for the victim to exit before (possibly) choosing a second process to kill. But the victim process cannot exit because it needs locks held by the allocating process. The system locks up and the owner of the system starts to seriously consider a switch to some version of BSD. 

When asked about this problem, XFS maintainer Dave Chinner quickly [wondered](<https://lwn.net/Articles/627432/>) why the memory-management code was resorting to the OOM killer rather than just failing the problematic memory allocation. The XFS code, he said, is nicely prepared to deal with an allocation failure; to him, using that code seems better than killing random processes and locking up the system as a whole. That is when memory management maintainer Michal Hocko [dropped a bomb](<https://lwn.net/Articles/627434/>) by saying: 

Well, it has been an unwritten rule that GFP_KERNEL allocations for low-order (<=PAGE_ALLOC_COSTLY_ORDER) never fail. This is a long ago decision which would be tricky to fix now without silently breaking a lot of code. Sad... 

The resulting explosion could be heard in [Dave's incredulous reply](<https://lwn.net/Articles/627435/>): 

We have *always* been told memory allocations are not guaranteed to succeed, ever, unless __GFP_NOFAIL is set, but that's deprecated and nobody is allowed to use it any more. 

Lots of code has dependencies on memory allocation making progress or failing for the system to work in low memory situations. The page cache is one of them, which means all filesystems have that dependency. We don't explicitly ask memory allocations to fail, we *expect* the memory allocation failures will occur in low memory conditions. We've been designing and writing code with this in mind for the past 15 years. 

A "too small to fail" allocation is, in most kernels, one of eight contiguous pages or less — relatively big, in other words. Nobody really knows when the rule that these allocations could not fail went into the kernel; it predates the Git era. As Johannes Weiner [explained](<https://lwn.net/Articles/627436/>), the idea was that, if such small allocations could not be satisfied, the system was going to be so unusable that there was no practical alternative to invoking the OOM killer. That may be the case, but locking up the system in a situation where the kernel is prepared to cope with an allocation failure also leads to a situation where things are unusable. 

One alternative that was mentioned in the discussion was to add the `__GFP_NORETRY` flag to specific allocation requests. That flag causes even small allocation requests to fail if the resources are not available. But, as Dave noted, trying to fix potentially deadlocking requests with `__GFP_NORETRY` is a game of Whack-A-Mole; there are always more moles, and they tend to win in the end. 

The alternative would be to get rid of the "too small to fail" rule and make the allocation functions work the way most kernel developers expect them to. Johannes's message included a patch moving things in that direction; it causes the endless reclaim loop to exit (and fail an allocation request) if attempts at direct reclaim do not succeed in actually freeing any memory. But, as he put it, ""the thought of failing order-0 allocations after such a long time is scary."" 

It is scary for a couple of reasons. One is that not all kernel developers are diligent about checking every memory allocation and thinking about a proper recovery path. But it is worse than that: since small allocations do not fail, almost none of the thousands of error-recovery paths in the kernel now are ever exercised. They could be tested if developers were to make use of the the kernel's [fault injection framework](<https://lwn.net/Articles/209257/>), but, in practice, it seems that few developers do so. So those error-recovery paths are not just unused and subject to bit rot; chances are that a discouragingly large portion of them have never been tested in the first place. 

If the unwritten "too small to fail" rule were to be repealed, all of those error-recovery paths would become live code for the first time. In a sense, the kernel would gain thousands of lines of untested code that only run in rare circumstances where things are already going wrong. There can be no doubt that a number of obscure bugs and potential security problems would result. 

That leaves memory-management developers in a bit of a bind. Causing memory allocation functions to behave as advertised seems certain to introduce difficult-to-debug problems into the kernel. But the status quo has downsides of its own, and they could get worse as kernel locking becomes more complicated. It also wastes the considerable development time that goes toward the creation of error-recovery code that will never be executed. Even so, introducing low-order memory-allocation failures at this late date may well prove too scary to be attempted, even if the long-term result would be a better kernel.
