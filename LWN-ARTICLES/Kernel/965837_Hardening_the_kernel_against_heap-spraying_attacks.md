---
title: "Hardening the kernel against heap-spraying attacks"
url: https://lwn.net/Articles/965837/
date: "March 21, 2024"
category: "Memory management-Slab allocators; Releases-6.11; Security-Kernel hardening"
author: "By Jonathan Corbet March 21, 2024"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 21, 2024

While a programming error in the kernel may be subject to direct exploitation, usually a more roundabout approach is required to take advantage of a security bug. One popular approach for those wishing to take advantage of vulnerabilities is [heap spraying](<https://en.wikipedia.org/wiki/Heap_spraying>), and it has often been employed to compromise the kernel. In the future, though, heap-spraying attacks may be a bit harder to pull off, thanks to [the "dedicated bucket allocator"](<https://lwn.net/ml/linux-kernel/20240305100933.it.923-kees@kernel.org/>) proposed by Kees Cook. 

Consider, for example, a use-after-free bug of the type that is, unfortunately, common in programs written in languages like C. Memory that is freed can be allocated to another user and overwritten; at that point, the code that freed the memory prematurely is likely to find an unpleasant surprise. The surprise will become even less endearing, though, if an attacker is able to control the data that is written into the freed memory. Often, that is all that is needed to turn a use-after-free bug into a full kernel compromise. 

It is, of course, difficult for an attacker to get their hands on precisely the chunk of memory that is being mishandled in the kernel. When precision is not possible, sometimes brute force will do. Heap spraying involves allocating as many chunks of memory as possible, and writing the crafted data into each, in the hope of happening upon the right one. Given the way that the kernel's slab allocator works, the chances of succeeding are higher than one might expect. 

The kernel has a variety of ways to allocate memory but, much of the time, a simple call to `kmalloc()` is used; this is especially true if the size of the memory to be allocated is not known ahead of time. Within the allocator, the requested memory size is rounded up to the nearest "bucket" size, and the requested chunk is allocated from the associated bucket. Those sizes are (mostly, but not exclusively) powers of two, so any allocation request between 33 and 64 bytes, for example, will be satisfied from the 64-byte bucket. 

If an attacker has determined that a given structure, allocated with `kmalloc()`, is used after being freed, they can attempt a heap-spraying attack by forcing the kernel to allocate (and write) a large number of objects from the same bucket. As it turns out, there are ways in which an attacker can get the kernel to do just that. 

The solution to this kind of problem (beyond fixing every use-after-free bug, of course) is to keep the allocation pools separate. If every allocation comes from its own heap, it cannot be used to spray somebody else's heap. Unfortunately, the current design exists for a reason; using common buckets for allocations across the kernel significantly increases allocation efficiency and memory utilization. So that is unlikely to change. 

During the 6.6 development cycle, an effort to improve the kernel's resistance to heap spraying, in the form of the [`kmalloc()` randomness patches](<https://lwn.net/Articles/938637/>), was merged. This work split each of the `kmalloc()` buckets 16 ways and caused the allocator to pick a random bucket (based on the request call site) to satisfy each request, with the result that a heap-spraying attack has a high chance of hitting the wrong bucket. It is a way of partially separating allocations without giving up entirely on a common set of heaps. Randomness hardens the kernel to an extent, but it is a probabilistic defense that will surely fail at times. 

The key point behind Cook's approach is that it is not necessary to separate all allocations into their own heaps; if the kernel could ensure that any _user-controllable_ allocation is satisfied from a different pool than anything else, heap spraying would be much harder to implement. To get there, a new API must be created for kernel subsystems that perform user-controllable allocations; that is what the patch series does. 

The first step for such a subsystem is to create its own heap for variable-size allocations with a call to: 

```
kmem_buckets *kmem_buckets_create(const char *name, unsigned int align,
    			  	      slab_flags_t flags,
    			  	      unsigned int useroffset,
    				      unsigned int usersize,
    			  	      void (*ctor)(void *));
```

This call is similar to [`kmem_cache_create()`](<https://elixir.bootlin.com/linux/v6.8.1/source/mm/slab_common.c#L361>), which creates a heap for fixed-size allocations. The name of the heap (which can be used for debugging) is given by `name`, and `align` describes the required alignment for objects allocated there. Flags for the slab allocator are given in `flags`. The `useroffset` and `usersize` parameters describe the portion of an object that might be copied to or from user space (information that is used by the kernel's [user-space copying hardening](<https://lwn.net/Articles/727322/>) mechanism), and `ctor()` is an optional constructor for allocated objects. The return value is a pointer to a `kmem_buckets` object that can be used for future allocations. 

Once that call succeeds, objects can be allocated with: 

```
void *kmem_buckets_alloc(kmem_buckets *b, size_t size, gfp_t flags);
```

Here, `b` is the pointer returned by `kmem_buckets_create()`, `size` is the size of the allocation, and `flags` is the usual [GFP flags](<https://lwn.net/Articles/920891/>); a pointer to the allocated memory is returned. A normal `kfree()` call can be used to free objects when they are (truly) no longer needed. There is also a `kmem_buckets_valloc()` that can fall back to `vmalloc()` if need be. 

The other part of the puzzle is to use this new allocator in the right places. Seemingly, the [`msgsnd()`](<https://man7.org/linux/man-pages/man3/msgsnd.3p.html>) system call is a favorite tool for heap-spraying attacks, since the kernel implements it by allocating a structure to contain the message to be sent (the size and contents of which are controlled by user space). Cook's series includes [a patch](<https://lwn.net/ml/linux-kernel/20240305101026.694758-8-keescook@chromium.org/>) causing `msgsnd()` to use the new bucket allocator, separating its allocations from all others and removing its utility in this kind of attack. [Another patch](<https://lwn.net/ml/linux-kernel/20240305101026.694758-9-keescook@chromium.org/>) switches the internal `memdup_user()` and `vmemdup_user()` functions, which are used to copy data from user space into the kernel. Many of the call sites for those functions will give user space some control over allocation sizes, so isolating them could prevent a lot of problems. 

While the proposed changes are relatively small, they could have an oversized impact on kernel security. Separating off user-controllable allocations in this way can block many of the exploits that have succeeded against the kernel in the past. Creating a kernel that is free of memory-safety bugs does not seem like a feasible goal in the near future, but making one where such bugs are harder to exploit is possible. Chances are that this patch series, in some form, will show up in the mainline before too long.
