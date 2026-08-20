---
title: "Per-call-site slab caches for heap-spraying protection"
url: https://lwn.net/Articles/986174/
date: "August 20, 2024"
category: "Memory management-Slab allocators; Security-Kernel hardening"
author: "By Jonathan Corbet August 20, 2024"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 20, 2024

One tactic often used by attackers set on compromising a system is [heap spraying](<https://en.wikipedia.org/wiki/Heap_spraying>); in short, the attacker fills as much of the heap as possible with crafted data in the hope of getting the target system to use that data in a bad way. If heap spraying can be blocked, attackers will lose an important tool. The kernel has some heap-spraying defenses now, including the [dedicated bucket allocator](<https://lwn.net/Articles/965837/>) merged for the upcoming 6.11 release, but its author, Kees Cook, thinks that more can be done. 

A heap-spraying attack can be carried out by allocating as many objects as possible and filling each with data of the attacker's choosing. If the kernel can be convinced to use that data, perhaps as the address of a function to call, then the attacker can gain control. Heap spraying is not a vulnerability itself, but it can ease the exploitation of an actual vulnerability, such as a use-after-free bug or the ability to overwrite a pointer. The kernel's `kmalloc()` function (along with several variants) allocates memory from the heap. Since `kmalloc()` is used heavily throughout the kernel, any call site that can be used for heap spraying can potentially be used to exploit a vulnerability in a distant, unrelated part of the kernel. That makes the `kmalloc()` heap a tempting target for attackers. 

`kmalloc()` makes its allocations from a set of "buckets" of fixed-sized objects; most (but not all) of those sizes are powers of two. So, for example, a 48-byte allocation request will result in the allocation of a 64-byte object. The structure behind `kmalloc()` is, in a sense, an array of heaps, each of which is used for allocations of a given size range. This separation can make heap spraying attacks easier, since it is not necessary to overwrite the entire heap to target an object of a given size. 

The dedicated bucket allocator creates a separate set of buckets for allocation sites that are deemed to present an especially high heap-spraying risk. For example, any allocation that can be instigated from user space and filled with user-supplied data would be a candidate for a dedicated set of buckets. Then, even if the attacker manages to thoroughly spray that heap, it will not affect any other allocations; the attacker's carefully selected data cannot be used to attack any other part of the kernel. 

The way to get the most complete protection from heap spraying would be to create a set of dedicated buckets for every `kmalloc()` call site. That would be expensive, though; each set of buckets occupies a fair amount of memory. Inefficiency at that level is the sort of tradeoff that kernel developers tend to view with extreme skepticism; creating a set of buckets for every call site simply is not going to happen. 

[This new patch series](<https://lwn.net/ml/all/20240809072532.work.266-kees@kernel.org>) from Cook is built around one of those observations that is obvious in retrospect: most `kmalloc()` call sites request objects of a fixed size that will never change. Often that size (the size of a specific structure, for example) is known at compile time. In such cases, providing the call site with a single dedicated slab for the size that is needed would give an equivalent level of protection against heap-spraying attacks. There is no need to provide buckets for all of the other sizes; they would never be used. 

The only problem with that idea is that there are thousands of `kmalloc()` call sites in the kernel. Going through and examining each one would be a tedious and possibly error-prone task, that would result in a lot of code churn. But the compiler knows whether the size parameter passed to any given `kmalloc()` call is a compile-time constant or not; all that is needed is a way to communicate that information to the call itself. If that information were accompanied by something that identified the call site, the slab allocator could set up dedicated slabs for the call sites where it makes sense. 

So the problem comes down to getting that information to `kmalloc()` in an efficient way. Cook's approach is an interesting adaptation of the [code-tagging framework](<https://lwn.net/Articles/932402/>) that was merged for the 6.10 release. Code tagging is part of the memory-allocation profiling subsystem, which is meant to help find allocation-related bugs; it ties allocations to the call site that requested them, so developers can find, for example, the source of a memory leak. 

Code tagging was not really meant as a kernel-hardening technology, but it does provide the call-site information needed here. Cook's series starts by augmenting the tag information stored for each call site with an indicator of whether the allocation size is constant and, if so, what that size is. That information will be available to the slab allocator when the `kmalloc()` call is made. 

If a given allocation request is at the `GFP_ATOMIC` level, it will be handled in the usual way to avoid adding any extra allocations to that path. Otherwise, though, the allocator will check whether that call site uses a constant size; if so, a dedicated slab will be created for that site and used to satisfy the allocation request (and all that follow). If the size is _not_ constant, then a full set of buckets will be created instead. Either way, the decision will be stored in the code tag to speed future calls. It is worth noting that this setup is not done for any given call site until the first call is made, meaning that it is not performed for the many `kmalloc()` call sites that will never execute in any given kernel. 

If this series is merged, the kernel will have three levels of defense against heap-spraying attacks. The [randomized slab option](<https://lwn.net/Articles/938637/>), merged for 6.6, creates 16 sets of slab buckets, then assigns each call site to one set randomly. Its memory overhead is relatively low, but the protection is probabilistic — it reduces the chance that an attacker can spray the target heap, but does not eliminate it. The dedicated-buckets option provides stronger protection, but is limited by the need to explicitly identify risky call sites and isolate them manually. This new option, instead, provides strong protection against heap spraying, but it will inevitably increase the memory overhead of the slab allocator. 

The amount of that overhead will depend on the workload being run. For an unspecified distribution kernel, Cook reported that the number of slabs reported in `/proc/slabinfo` grew by a factor of five or so. Should the series land in the mainline, it will be up to distributors to decide whether to enable this option or not. When a kernel is going to run on a system that is at high risk of heap-spraying attacks, though, that may prove to be an easy decision to make.
