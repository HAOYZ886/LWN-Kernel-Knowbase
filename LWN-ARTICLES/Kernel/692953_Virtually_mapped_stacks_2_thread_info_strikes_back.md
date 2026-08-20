---
title: "Virtually mapped stacks 2: thread_info strikes back"
url: https://lwn.net/Articles/692953/
date: "June 29, 2016"
category: Kernel stack
author: "By Jonathan Corbet June 29, 2016"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 29, 2016

In [last week's episode](<https://lwn.net/Articles/692208/>), Andy Lutomirski had posted a set of patches moving kernel stacks to the kernel's `vmalloc()` area. There are a number of advantages to doing so, including elimination of some high-order memory allocations, improved security, and better diagnostic output when a stack overflow occurs. There was just one little problem: an additional 1.5µs of kernel overhead during process creation, a cost that Linus was unwilling to accept. The first attempt to address that cost ran afoul of an obscure kernel data structure, but the end result looks to be a substantial cleanup of the kernel's management of process information. 

The performance regression comes from using `vmalloc()` to allocate kernel stacks; `vmalloc()` is a relatively expensive way to allocate memory and has not been subjected to the same sort of optimization work that the slab allocators have seen. One suggestion that had been made was to cache a small number of kernel stacks, allowing the quick reuse of cache-hot stacks after processes exit. The hope was that, by eliminating `vmalloc()` calls, the code could be made faster in general. 

#### A useless cache

Andy went off to implement this idea and [reported](<https://lwn.net/Articles/692955/>) a discouraging result: ""I implemented a percpu cache, and it's useless."" The problem, in short, is that a process's resources (including its kernel stack) are not cleaned up immediately when the process exits. Instead, the read-copy-update (RCU) mechanism is used to ensure that no references to those resources remain before they are freed. That means (1) the freeing of the kernel stack will be delayed until the end of the next RCU grace period, and (2) the resources for _all_ processes that exited during that grace period will be freed together. So the cache for kernel stacks will almost always be empty, then will occasionally be hit with large numbers of released stacks, most of which will not fit into the cache and, thus, will be simply freed. In other words, the cache hit rate, especially with fork-heavy loads, will be close to zero. 

In theory there should be no need for a process's kernel stack after that process has died, so one might think that the stack could be released immediately, even if the other data structures need to stay around. The problem is that the core information the kernel maintains about processes lives in two different places: 

  * The massive [`task_struct`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/include/linux/sched.h#n1457>) structure, found in `<linux/sched.h>`. This structure, which is (modulo a disturbing number of `#ifdef` blocks) architecture-independent, contains most of the information the kernel needs to know about a running process. 

  * The small [`thread_info`](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/thread_info.h#n45>) structure, which is architecture-specific. 

The `task_struct` structure is allocated from the heap like most other kernel data structures. The `thread_info` structure, though, lives at the bottom of the kernel stack, making it impossible to reuse the kernel stack as long as something might try to reference that structure. For a brief period, Linus pursued changes that would allow the `thread_info` structure to be freed quickly, even while the `task_struct` structure persisted, but it quickly became clear that no easy solutions were to be found in that direction. Some information in `thread_info`, the `flags` field in particular, can be accessed at surprising times and needs to remain available as long as the kernel has any record of the associated process at all. 

The existence of these two structures is something of a historical artifact. In the early days of Linux, only the `task_struct` existed, and the whole thing lived on the kernel stack; as that structure grew, though, it became too large to store there. But placement on the kernel stack conferred a significant advantage: the structure could be quickly located by masking some bits out of the stack pointer, meaning there was no need to dedicate a scarce register to storing its location. For certain heavily used fields, this was not an optimization that the kernel developers wanted to lose. So, when the `task_struct` was moved out of the kernel-stack area, a handful of important structure fields were left there, in the newly created `thread_info` structure. The resulting two-structure solution is still present in current kernels, but it doesn't necessarily have to be that way. 

#### Getting rid of thread_info

In relatively recent times, the kernel has moved to using per-CPU variables to store many types of frequently-needed information. The scheduler caches some crucial process-related information about the currently running process in the per-CPU area; that turns out to be faster than looking at the bottom of the kernel stack. So the amount of useful data stored in `struct thread_info` has decreased over time. Given that, an obvious question comes to mind: could the `thread_info` structure be removed altogether? The question is not exactly new; moving `struct thread_info` away from the kernel stack was one of Andy's objectives from the beginning. But the performance issue gave this question a higher level of urgency. 

Linus quickly [discovered](<https://lwn.net/Articles/692957/>) that some users of the `thread_info` structure have no need of it, even with no other changes. Mostly they used it to find the `task_struct` structure — which, typically, they already had a pointer to. He fixed those, and committed the result for the [4.7-rc5](<https://lwn.net/Articles/692762/>) release. This kind of change might not qualify as the sort of bug fix that late-cycle patches are normally restricted to, but he [made it clear](<https://lwn.net/Articles/692958/>) that he thought they were an acceptable change: ""Those are the 'purely legacy reasons for a bad calling convention', and I'm ok with those during the rc series to make it easier for people to play around with this."" 

He pushed the work further, getting to a point where he could move the (somewhat reduced) `thread_info` structure off the stack, and embed it within the `task_struct` instead. That work progressed to where it would boot on a test system; Andy then picked it up and integrated it into his larger patch series. 

As of this writing, that series is in [its fourth revision](<https://lwn.net/Articles/693026/>). It moves many `thread_info` fields into `task_struct`, changing the users of those fields along the way. At the end, the `thread_info` structure, now containing only the `flags` field, is moved over to the `task_struct` as well. Getting there requires a number of changes to the low-level architecture code, so it is an x86-only change at the moment. It seems likely that other architectures will follow along, though; getting rid of the separate `thread_info` structure is a useful cleanup and security enhancement, even without the rest of the work. 

With regard to the real objective of the patch set (moving kernel stacks to the `vmalloc()` area): the removal of the `thread_info` structure makes it possible to free the kernel stack as soon as the owning process exits — no RCU grace period required. That, in turn, makes it sensible to add a small per-CPU cache holding up to two free kernel stacks. With the cache, Andy says, the 1.5µs performance regression becomes a 0.5-1µs performance gain. 

So, at this point, Andy has a patch series that simplifies some core code, provides immediate detection of kernel-stack overruns, gives better diagnostics when that occurs, improves the security of the kernel, and even makes things run faster. Unsurprisingly, objections are now becoming difficult to find. The only remaining question might be when to merge this code, and the answer appears to be during the 4.8 merge window. That is arguably aggressive, given the fundamental nature of the changes and the fact that there must certainly be a surprise or two still lurking there; indeed, [one such surprise](<https://lwn.net/Articles/693021/>) is still being worked out as of this writing. But the 4.8 cycle should give time to work through those surprises, and the end result should be worth having.
