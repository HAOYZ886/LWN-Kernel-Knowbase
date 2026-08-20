---
title: Goodbye to GFP_TEMPORARY and dma_alloc_noncoherent()
url: https://lwn.net/Articles/732107/
date: "August 28, 2017"
category: "Direct memory access; Memory management-GFP flags"
author: "By Jonathan Corbet August 28, 2017"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 28, 2017

Like most actively developed programs, the kernel grows over time; there have only been two development cycles ever (2.6.36 and 3.17) where the kernel as a whole was smaller than its predecessor. The kernel's internal API tends to grow in size and complexity along with the rest. The good thing about the internal API, though, is that it is completely under the control of the development community and can be changed at any time. Among other things, that means that parts of the kernel's internal API can be removed if they are no longer needed — or if their addition in the first place is deemed to be a mistake. A pair of pending removals in the memory-management area shows how this process can work. 

#### GFP_TEMPORARY

One of the many challenges faced by the kernel's memory-management subsystem is fragmentation. If allocations are not placed carefully, the system's free memory can end up split into many small chunks that cannot be coalesced; that can lead to allocations failing even though much of the system's memory is idle. This is particularly true of memory allocations for use by the kernel itself. Those allocations can be long-lived and there is usually no way to relocate them if they are in the way. A single small allocation can prevent the reuse of an entire page; that, in turn, can block the creation of larger chunks of memory around that page. 

It has long been understood that not all kernel memory allocations are equal. Some data structures are critical to the operation of the system and cannot be removed; consider the structures describing a mounted filesystem or a running process, for example. Others, though, exist to improve the system's performance and can be dropped if needed; the inode and dentry caches in the virtual filesystem layer are perhaps the biggest examples of this type of structure. The latter type of structure is called "reclaimable". 

A key heuristic used within the memory-management subsystem is to try to separate reclaimable and non-reclaimable allocations. A page full of reclaimable allocations can, in theory at least, be recovered for other uses when memory is tight. But a single non-reclaimable allocation will prevent the reuse of the entire page. Separating the two types increases the probability that pages containing reclaimable allocations can, in truth, be reclaimed should the need arise. 

Back in 2007, Mel Gorman [added the `GFP_TEMPORARY` allocation type](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e12ba74d8ff3e2f73a583500d7095e406df4d093>) in an attempt to make memory allocation more flexible. The reasoning was this: some memory allocations last a long time, while others are highly transient. A structure allocated to represent a newly added device may persist for the lifetime of the system, while memory allocated to satisfy a system call may be returned within milliseconds. When an allocation is short-lived, it doesn't matter whether it is reclaimable or not; since it will be returned shortly regardless, it is unlikely to hold up the reclaim of a page full of otherwise reclaimable allocations. So `GFP_TEMPORARY` allocations were allowed to draw from the reclaimable pool, even though there was no mechanism by which they could be reclaimed. 

Earlier this year, `GFP_TEMPORARY` was [the subject of an extensive discussion](<https://lwn.net/Articles/713076/>) that was, at the start, focused on a seemingly simple question: what does "temporary" mean? Is there a limit on how long the allocation can be held? Is the holder of a `GFP_TEMPORARY` allocation allowed to block or take locks? It turns out that this was not specified when `GFP_TEMPORARY` was added. The discussion failed to fill that void, and a review of the `GFP_TEMPORARY` call sites in the kernel revealed some decidedly non-temporary uses. It became clear that nobody really knew what a "temporary" allocation was supposed to be. 

There was talk of trying to nail down that definition, but Michal Hocko pushed a different approach: [remove `GFP_TEMPORARY` entirely](<https://lwn.net/Articles/729145/>). The current uses, he said, did not justify keeping it around: 

I have checked some random users and none of them has added the flag with a specific justification. I suspect most of them just copied from other existing users and others just thought it might be a good idea to use without any measuring. This suggests that GFP_TEMPORARY just motivates for cargo cult usage without any reasoning. 

There were a handful of complaints about the loss of the flag, but no serious opposition to the change. Other developers, [including Neil Brown](<https://lwn.net/Articles/732108/>), agreed with the change: 

If we have a flag that doesn't have a well defined meaning that actually affects behavior, it will not be used consistently, and if we ever change exactly how it behaves we can expect things to break. So it is better not to have a flag, than to have a poorly defined flag. 

He suggested improving the kernel's notion of reclaimability of allocations instead. That may happen in the future, but the removal of `GFP_TEMPORARY` is set to happen more quickly. The patches are in linux-next now, meaning they are on track to hit the mainline during the 4.14 merge window. Should that happen, `GFP_TEMPORARY` will itself prove to have been temporary — for a "ten years" value of "temporary". 

#### dma_alloc_noncoherent()

The allocation of memory for direct memory access (DMA) operations is not as simple as it might seem. Devices often have a different view of memory than the CPU does, and allocations for DMA must bridge that gap. These allocations must usually be physically contiguous, and they have to be in a region of memory that the target device is able to access, for example. An interesting additional requirement is handled by `dma_alloc_noncoherent()`: 

```
void *dma_alloc_noncoherent(struct device *dev, size_t size,
        				dma_addr_t *dma_handle, gfp_t flag);
```

A call to `dma_alloc_noncoherent()` is an explicit request to allocate a DMA buffer in a noncoherent region of memory. Memory that is cache-coherent looks the same to both the CPU and I/O devices. If the CPU writes to that memory, its writes will be visible to the device; similarly, if the device writes a region of memory, the CPU will immediately see the new data. Noncoherent memory lacks that guarantee; if the CPU wants to read data placed into memory via a DMA operation, it must take care to invalidate its own memory caches after the completion of the I/O operation, but before its first access. 

Noncoherent memory is clearly trickier to work with; without sufficient care, it is easy to end up with corrupted data. So one might wonder why anybody would want to ask for it specifically. The answer is that on architectures where cache coherence doesn't come naturally (ARM, for example), coherent memory is far slower. Turning on coherence generally involves turning off caching, with a predictable effect on performance. For situations involving any significant data processing, using coherent memory is just not an option. 

It is thus important to be able to allocate noncoherent memory for DMA buffers, which raises the question of why Christoph Hellwig is [working to remove `dma_alloc_noncoherent()`](<https://lwn.net/Articles/732109/>). The answer is that, on any reasonably current system, control over memory access modes is more sophisticated than simply turning caching on or off. Memory can be configured to allow write combining (where multiple write operations can be grouped by the hardware for performance), for example, or it can be set to allow operations to be reordered. Many of these features can be configured together. Creating new allocation function for each combination is clearly unlikely to lead to joy, so the kernel developers added a new set of functions in the 3.4 development cycle, including: 

```
void *dma_alloc_attrs(struct device *dev, size_t size, dma_addr_t *dma_handle,
    			  gfp_t flag, unsigned long attrs);
```

The `attrs` field can be used to specify [a whole range of attributes](<https://lwn.net/Articles/732110/>), including `DMA_ATTR_NON_CONSISTENT` to obtain a noncoherent mapping. This function clearly fills the same role as `dma_alloc_noncoherent()` and a lot more besides, so there is little reason to keep `dma_alloc_coherent()` around. Hellwig has been working to remove it, which means updating all of its callers to use `dma_alloc_attrs()` instead. Much of that work went in during the 4.13 merge window; only three call sites remain. His current patches remove those last three, along with the function itself. Any out-of-tree drivers using `dma_alloc_noncoherent()` will have to be updated separately, of course. 

In both cases, the kernel's internal API is getting (slightly) smaller, but no functionality is being lost. This work is an example of the sort of cleanups that are possible when there is no need to maintain API compatibility. Interfaces exposed to user space must be preserved, but the ability to evolve internally is a big part of why the kernel remains maintainable despite having just celebrated its 26th birthday.
