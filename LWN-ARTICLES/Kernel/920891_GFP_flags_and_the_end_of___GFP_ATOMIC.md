---
title: GFP flags and the end of __GFP_ATOMIC
url: https://lwn.net/Articles/920891/
date: "January 27, 2023"
category: "Memory management-GFP flags; Releases-6.3"
author: "By Jonathan Corbet January 27, 2023"
---

By **Jonathan Corbet**  
January 27, 2023

Memory allocation within the kernel is a complex business. The amount of physical memory available on any given system will be strictly limited, meaning that an allocation request can often only be satisfied by taking memory from somebody else, but some of the options for reclaiming memory may not be available when a request is made. Additionally, some allocation requests have requirements dictating where that memory can be placed or how quickly the allocation must be made. The kernel's memory-allocation functions have long supported a set of "GFP flags" used to describe the requirements of each specific request. Those flags will probably undergo some changes soon as the result of [this patch set](<https://lwn.net/ml/linux-kernel/20230113111217.14134-1-mgorman@techsingularity.net/>) posted by Mel Gorman; that provides an opportunity to look at those flags in some detail. 

The "GFP" in GFP flags initially stood for "get free page", referring to [`__get_free_pages()`](<https://elixir.bootlin.com/linux/v6.2-rc5/source/mm/page_alloc.c#L5595>), a longstanding, low-level allocation function in the kernel. GFP flags are used far beyond that function, but it's worth remembering that they are relevant to _full-page_ allocations. Functions (like `kmalloc()`) that allocate smaller chunks of memory may take GFP flags, but they are only used when those functions must get full pages from the memory-management subsystem. 

Most developers see GFP flags in the form of macros like `GFP_ATOMIC` or `GFP_KERNEL`, but those macros are actually constructs made up of lower-level flags. Thus, for example, in the 6.2-rc kernels, `GFP_ATOMIC` is [defined as](<https://elixir.bootlin.com/linux/v6.2-rc5/source/include/linux/gfp_types.h#L332>): 

```
#define GFP_ATOMIC	(__GFP_HIGH|__GFP_ATOMIC|__GFP_KSWAPD_RECLAIM)
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Each of the component flags modifies the request in some way; `__GFP_HIGH` marks it as a "high priority" request, for example, with implications that will be described below. Back in 2021, though, Neil Brown observed that ""__GFP_ATOMIC serves little purpose"" and [posted a patch](<https://lwn.net/ml/linux-kernel/163712397076.13692.4727608274002939094@noble.neil.brown.name/>) to remove it. That patch languished for over a year, though it did inspire a slow-moving conversation on the meaning of certain GFP flags. In October, Andrew Morton [threatened](<https://lwn.net/ml/linux-kernel/20221016193827.37c8c2165b22ae4401c269b4@linux-foundation.org/>) to drop it. That inspired Gorman to pick it up and fold it into the current series, which makes a number of changes to how the low-level GFP flags work. 

#### Low-level GFP flags

These flags are defined in [`include/linux/gfp_types.h`](<https://elixir.bootlin.com/linux/v6.2-rc5/source/include/linux/gfp_types.h>). There are multiple variants of each flag. Thus, for example, `__GFP_ATOMIC` is defined as: 

```
#define __GFP_ATOMIC	((__force gfp_t)___GFP_ATOMIC)
```

While the three-underscore `___GFP_ATOMIC` is simply defined as: 

```
#define ___GFP_ATOMIC		0x200u
```

The intermediate (two-underscore) level of flags is there to enable type checking for the GFP-flags argument to a number of functions, while the three-underscore level allows easy manipulation within the memory-management subsystem. Developers outside of that subsystem should not use the three-underscore versions, but they are the ones that, in the end, define the options that are available with memory-allocation requests. 

So, for the curious, here is the set of low-level GFP flags and their effect on allocation requests after the application of Gorman's patch set, grouped into a few broad categories. 

#### Placement options

A number of GFP flags affect where an allocation is located in physical memory: 

`**___GFP_DMA**`
     This is an ancient flag reflecting the limitations of early x86 systems, which could only support 24-bit DMA addresses on the ISA bus; it caused the allocation to be placed in the lowest 16MB of physical memory. Even the worst hardware supported on current systems should not suffer from this limitation but, as `gfp_types.h` notes, it cannot be easily removed: ""careful auditing"" would be required. 
`**___GFP_DMA32**`
     Like `___GFP_DMA`, this flag is for devices with a limited DMA range; this one restricts the allocation to physical memory that is addressable with 32 bits. This flag, hopefully, is only needed with older hardware that had a hard time making the 64-bit transition. 
`**___GFP_HIGHMEM**`
     This flag indicates that "high memory" can be used to satisfy the allocation request. High memory is [described in this article](<https://lwn.net/Articles/813201/>); it only exists on 32-bit systems where it is not possible to map all of physical memory into the kernel's address space. There is no high memory on 64-bit systems, so this flag has no effect there. 
`**___GFP_MOVABLE**`
     Indicates memory that can be moved by the memory-management subsystem if needed to help the reclaim process. User-space pages, for example, are movable since they are accessed via page tables that can be updated to a new location without the owning process noticing. 
`**___GFP_RECLAIMABLE**`
     This flag indicates slab memory that can be reclaimed via shrinkers when resources get tight. The memory-management subsystem tries to keep both movable and reclaimable allocations together in the same memory zones to facilitate the freeing of larger ranges of memory when needed. 
`**___GFP_HARDWALL**`
     This flag disallows the allocation of memory outside of the memory nodes in the calling process's cpuset. 
`**___GFP_THISNODE**`
     Allocations with this flag can only be satisfied by memory that is on the current NUMA node. 

#### Access to reserves

The memory-management subsystem works hard to keep a reserve of free memory at all times. Freeing memory can often require allocating memory first — to set up an I/O transfer to write the contents of dirty pages to persistent storage, for example — and things go badly indeed if that allocation fails. There are a few options describing whether an allocation can eat into the reserves, and by how much. 

`**___GFP_HIGH**`
     "High-priority" allocations are marked with this flag. What this means in practice, as stabilized by Gorman's patch set, is that the allocation is allowed to dip into the memory reserves that the kernel keeps for important allocations. With this flag, a request is allowed to deplete the reserves down to 50% of their normal size. 
`**___GFP_MEMALLOC**`
     An allocation with this flag will bypass all limits on use of reserves, grabbing any chunk of memory that is available. It is only intended to be used in cases where the allocation is done with the purpose of making more memory available in the near future. 
`**___GFP_NOMEMALLOC**`
     Explicitly disables any access to memory reserves. This flag was initially introduced to prevent memory pools from running down the kernel's reserves. 
Note that `___GFP_KSWAPD_RECLAIM`, described below, also has an effect on the use of reserves. 

#### Side effects

An allocation request made from atomic context cannot be allowed to block, and requests made from a filesystem should not cause a recursive call back into that filesystem. A few of the defined GFP flags reflect these constraints, describing what the memory-management subsystem is allowed to do to satisfy a request: 

`**___GFP_IO**`
     Requests with this flag are allowed to start I/O if needed to reclaim memory. It will be present for "normal" requests, but missing for requests made from within the storage layer, for example, where recursive I/O operations should be avoided. 
`**___GFP_FS**`
     This flag allows the request to call into the filesystem layer if needed to reclaim memory; like `___GFP_IO`, it is used (by its absence) to avoid recursive calls when the filesystem itself is allocating memory. 
`**___GFP_DIRECT_RECLAIM**`
     Allows the allocation call to enter direct reclaim, meaning that the calling thread can, itself, be put to work freeing memory to satisfy the allocation. Direct reclaim increases the chances of the allocation succeeding, but can also increase the latency of the request and may cause the calling thread to block. 
`**___GFP_KSWAPD_RECLAIM**`
     This flag allows the `kswapd` process to be invoked to perform reclaim. That is normally desired, but there are cases where having `kswapd` running could interfere with other memory-management operations. Less obviously (but perhaps more importantly), this flag also indicates that the allocation request should not block for any reason; indeed, `GFP_NOWAIT` is a synonym for this flag. If `___GFP_HIGH` is also set, this flag will allow access to 62.5% of the memory reserves. 

#### Warnings and retries

Yet another set of options describes what should be done if an initial attempt to fulfill an allocation request fails: 

`**___GFP_NOWARN**`
     Prevents the printing of warnings in the system log should the request fail. This flag is used in cases where failures may be expected and there is a readily available workaround available; an example would be an attempt to allocate a large, contiguous area where it is possible to get by with a lot of smaller allocations if need be. 
`**___GFP_RETRY_MAYFAIL**`
     Indicates a request that is important and which can wait for additional retries if the first attempt at allocation fails. 
`**___GFP_NOFAIL**`
     Marks a request that cannot be allowed to fail; the memory-management subsystem will retry indefinitely in this case. There have been [occasional attempts to remove this flag](<https://lwn.net/Articles/401915/>) on the theory that all kernel code should be able to handle allocation failures, but there are still quite a few uses of it. 
`**___GFP_NORETRY**`
     This flag will cause an allocation request to fail quickly if memory is not readily available. It is useful in places where allocation failures can be handled relatively easily and it is better to not stress the system by trying harder to reclaim memory. 

#### Miscellaneous flags

Finally, there is the inevitable set of flags that don't fit into any other category: 

`**___GFP_ZERO**`
     This flag requests that the requested page(s) be zeroed before being returned. 
`**___GFP_WRITE**`
     Indicates that the page will be written to soon. This flag is only observed in a couple of spots. One is to try to spread to-be-written pages across memory zones. The other is a tweak to the "refault" code that handles a process's working set; if a previously reclaimed file page is brought back in to be rewritten, it is not treated as part of the working set. 
`**___GFP_COMP**`
     A multi-page allocation should return a [compound page](<https://lwn.net/Articles/619514/>). 
`**___GFP_ACCOUNT**`
     This flag causes the allocation to be charged to the current process's control group. It is only used for allocations to be used within the kernel; user-space pages are already accounted in this way. 
`**___GFP_ZEROTAGS**`
     Causes the internal "tags" metadata associated with the page to be cleared if the page itself is being zeroed on allocation. This is a minor optimization that is used in exactly one place in the arm64 page-fault-handling code. 
`**___GFP_SKIP_ZERO**`
`**___GFP_SKIP_KASAN_UNPOISON**`
`**___GFP_SKIP_KASAN_POISON**`
     These three flags are used to tweak the checking by the KASAN sanitizer. 
`**___GFP_NOLOCKDEP**`
     Disables checking of the allocation context done by the lockdep locking checker. This flag is only used within the memory-management subsystem and XFS filesystem. 

#### In conclusion

As can be seen, there are a lot of ways to modify how memory allocation is done in the kernel. After Gorman's patch series, `GFP_ATOMIC` still exists (there are over 5,000 call sites in the kernel, after all), but it is defined as: 

```
#define GFP_ATOMIC	(__GFP_HIGH|__GFP_KSWAPD_RECLAIM)
```

So, from the list above, we see that a `GFP_ATOMIC` allocation request will never block, and it has deep access to the kernel's memory reserves. That is why use of `GFP_ATOMIC` is discouraged in any situation where it is not truly necessary. 

One other change made in this patch set is to treat allocations made by realtime tasks as being implicitly marked as `___GFP_HIGH`, since dipping into the reserves is seen as better than delaying the task and causing it to miss a deadline. But, as Gorman [pointed out](<https://lwn.net/ml/linux-kernel/20230113111217.14134-3-mgorman@techsingularity.net/>), if access to reserves is needed, the system is under memory pressure and chances are good that the deadlines aren't going to work out anyway. The patch warns that the special-case for realtime tasks will be removed at some point in the future. 

This patch series is in its third iteration, not counting Brown's initial posting. The discussion seems to have slowed down, so it is looking like it is close to ready to head into the mainline. Then `___GFP_ATOMIC` will be no more, some of the other flags will be a bit better defined — and most developers presumably will not even notice.
