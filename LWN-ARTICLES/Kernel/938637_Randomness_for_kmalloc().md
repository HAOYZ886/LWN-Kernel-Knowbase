---
title: Randomness for kmalloc()
url: https://lwn.net/Articles/938637/
date: "July 24, 2023"
category: "Memory management-Slab allocators; Releases-6.6; Security-Kernel hardening"
author: "By Jonathan Corbet July 24, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 24, 2023

The kernel's address-space layout randomization is intended to make life harder for attackers by changing the placement of kernel text and data at each boot. With this randomization, an attacker cannot know ahead of time where a vulnerable target will be found on any given system. There are techniques, though, that can be effective without knowing precisely where a given object is stored. As a way of hardening systems against such attacks, the kernel will be gaining yet another form of randomization. 

"[Heap spraying](<https://en.wikipedia.org/wiki/Heap_spraying>)" attempts to fill the target system's heap with known data; it generally works by allocating large amounts of memory and filling it with the data of interest. A successful attack can fill much of the heap with a known pattern. If the target system can then be convinced to dereference an invalid pointer into the heap, chances are good that the access will land on attacker-controlled data. 

As an example, consider a use-after-free vulnerability. Once an object has been freed, it can be allocated by the attacker and overwritten. Then it is just a matter of waiting for the targeted code to use its (now invalid) pointer, and the game is over; a vulnerability that simply let an attacker allocate and write some memory has been escalated into something rather more severe. Needless to say, attackers find that kind of capability attractive; heap spraying has been used in a number of successful exploits. 

The heap, in the kernel, is a bit of a nebulous concept; one could think of it as comprising almost all of the physical memory in the system. But much of the memory management within the kernel is handled by the slab allocator. In theory, there are hundreds of independent slabs in the Linux kernel; `/proc/slabinfo` shows nearly 300 on your editor's system. This number reflects a large number of in-kernel users of slabs — subsystems that each need to allocate objects of different sizes. That separation should help to thwart heap-spraying attacks, since the ability to spray one slab should leave the others unaffected. An attacker cannot spray a heap that they cannot allocate from. 

In practice, though, there is not as much isolation between users as it might seem. Calls to `kmalloc()`, which are how much of the memory in the kernel is allocated, all share a set of common slabs. There are tens of thousands of calls to `kmalloc()` (and variants like `kzalloc()`) in the kernel, so memory obtained that way is a fairly obvious target for spraying attacks. The slab allocator will also often merge slabs containing objects of similar sizes, again putting multiple users into the same "heaps". In summary, the Linux kernel, too, can be vulnerable to heap-spraying attacks. 

As Ruigi Gong notes in [this patch](<https://lwn.net/ml/linux-kernel/20230714064422.3305234-1-gongruiqi@huaweicloud.com/>), separating those slab users to thwart spraying attacks is not a practical alternative. All of those `kmalloc()` users are not going to change, and turning off slab merging entirely would have a performance cost. Instead, a lot of protection could, in theory, be had if all of those `kmalloc()` users were somehow separated from each other, at least to a degree. 

The `kmalloc()` allocator uses a set of a dozen or so slabs for objects of different sizes; on your editor's system, the smallest is for eight-byte objects, while the largest is for 8KB objects, with the size (usually) doubling from one slab to the next. When `kmalloc()` is called, the requested size is rounded up to the nearest slab size, and the allocation is made from that slab. So, for example, a 36-byte request will result in the allocation of 64 bytes of memory. Just to make things more complicated, there are actually four sets of slabs as described here: one for "normal" allocations, one for memory to be used for DMA, one for memory marked as reclaimable (allocated with `__GFP_RECLAIMABLE`), and one for non-reclaimable memory charged to control groups. 

Gong's patch set adds another dimension to this matrix by adding another 15 slabs for each size — but only for "normal" allocations. Whenever a `kmalloc()` call falls into the normal category (which is most of the time), one of the 16 slabs for the appropriate size will be chosen at random, and the allocation will be made from that slab. In this way, it is hoped, any memory that can be sprayed by an attacker will be separated from the memory used by the vulnerable code that is under attack. 

To raise the chances that things turn out that way, some thought has gone into the selection of the random slab for the allocations. There are two values that are used in this calculation: a random seed generated at boot time and the address from which `kmalloc()` was called. As a result, any given `kmalloc()` call site will always allocate from the same slab, but the specific slab will vary from one boot to the next. So, for an attacker, it is not just a matter of performing more allocations to spray all of the relevant slabs; instead, a call that hits the correct slab for the current boot cycle must be found. It is not an absolute defense, but splitting the slabs in this way raises the bar for a successful attack considerably. 

Benchmarks included with the patch show a small performance overhead and a bit of increased memory use when this feature is enabled; the cost seems low enough that it would not be noticed by most users. In response to a previous post, Kees Cook had [remarked](<https://lwn.net/ml/linux-kernel/202305161204.CB4A87C13@keescook/>) that it provided ""a nice balance"" between the various options that are available for hardening against heap-spraying attacks. The fifth revision, posted on July 14, was quickly [applied](<https://lwn.net/ml/linux-kernel/d9f96152-e48e-7a1f-cd00-b7d508c5560f@suse.cz/>) by slab maintainer Vlastimil Babka; this work is now in linux-next and appears set to enter the mainline during the 6.6 merge window.
