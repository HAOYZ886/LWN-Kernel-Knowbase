---
title: A handful of DMA topics
url: https://lwn.net/Articles/90870/
date: "June 23, 2004"
category: Direct memory access; dma get required mask
author: 
---

The [generic DMA layer](<https://lwn.net/Articles/28092/>) provides a way for device drivers to allocate and work with direct memory access regions without regard for how the underlying hardware does things. This interface works well, for the most part, but, as with the rest of the kernel, occasional issues come up. Here's a few that were discussed over the last week. 

Many devices can perform full 64-bit DMA operations. This capability is nice on large-memory systems, but working with larger addresses can also bring a performance penalty. As a way of helping drivers pick the optimal size for DMA address descriptors, James Bottomley has [proposed](<https://lwn.net/Articles/89971/>) the creation of a new function called `dma_get_required_mask()`. 

The current API already has `dma_set_mask()`, which tells the kernel about the range of DMA addresses the device can access. The new function would be called after an invocation of `dma_set_mask()`; it would return a new bitmask describing what the platform sees as the optimal set of DMA addresses, taking the device's original DMA mask into account. If the specific hardware situation does not require the use of larger addresses, the platform can suggest using the faster, 32-bit mode even when the device can handle larger addresses. The driver can then use that advice to set a new mask describing what it will actually use. 

The "scatterlist" mechanism is another part of the DMA subsystem; it allows drivers to set up scatter/gather I/O, where the buffer to be transferred is split into multiple, distinct chunks of memory. Scatter/gather is useful in a number of situations, including network packets (which are assembled from multiple chunks), the `readv()` and `writev()` system calls, and for I/O directly to or from user-space buffers, which can be spread out in physical memory. The mapping functions for scatter/gather I/O will coalesce pieces of the buffer which turn out to be physically adjacent in memory. In practice, that has turned out not to happen very often; one recent [report](<https://lwn.net/Articles/90876/>) showed that, out of approximately 32,000 segments, all of 40 had been merged in this manner. 

It turns out, however, that the Linux memory allocator is not helping the situation. When the allocator breaks up a large block of pages to satisfy smaller requests (a frequent occurrence), it returns the highest page in the block. A series of allocations will, thus, obtain pages in descending order. If those pages are assembled into an I/O buffer, each page will need to be a separate segment in a scatter/gather operation, since the reverse-allocated pages cannot be merged. 

William Lee Irwin put together [a patch](<https://lwn.net/Articles/90877/>) which causes the allocator to hand out pages from the bottom of a block instead of the top. With this patch applied, the merge rate in this particular test went up to over 55%. Larger segments lead to faster I/O setup and execution, which is a good thing. Sometimes a tiny patch can make a big difference, once you know where the problem is. 

Meanwhile, Ian Molton [turned up](<https://lwn.net/Articles/90878/>) a different sort of problem. Some types of interfaces have their own onboard memory. This memory is, often, accessible to the CPU, and it can be used by devices attached to the interface for DMA operations. But that memory is not part of the regular system RAM, and it typically does not show up in the system memory map. As a result, the generic DMA functions will not make use of this memory when allocating DMA buffers. 

It would be nice to be able to make use of this memory, however. It is there, and it can be used to offload some DMA buffers from main memory. On some systems, it may be the only memory which is usable for DMA operations to certain devices. The DMA API has even been set up with this sort of memory in mind; it can handle cases where, for example, the memory in question has a different address from the device's point of view than it does for the processor. It would seem that the addition of an architecture-specific module to the DMA API could enable such memory to be allocated on platforms which have it, when the DMA target is a device which can make use of it. 

The biggest problem would appear to be that this sort of remote memory is not part of the system's memory map, and, thus, there is no `struct page` structure which describes it. The lack of a `page` structure makes certain macros fail. It also completely breaks any driver which tries to map the buffer into user space via the `nopage()` VMA operation. And, it turns out, drivers really do that; the ALSA subsystem, for example, maps buffers to user space in this manner. 

Once a problem is identified, it can usually be fixed. The right approach in this case would appear to be a combination of two things. The first is to simply fix any bad assumptions in drivers with regard to how they can treat DMA buffers. If the driver expects that a `page` structure exists for a DMA buffer, it is broken and simply needs to be fixed. The second part is to provide an architecture-independent way for device drivers to map DMA buffers into user space. 

To that end, Russell King has [proposed](<https://lwn.net/Articles/90880/>) yet another DMA API function: 

```
int dma_map_coherent(struct device *dev, 
                             struct vm_area_struct *vma,
                             void *cpu_addr,
                             dma_addr_t handle,
                             size_t size);
```

This function would take the given mapped DMA buffer (as described by `cpu_addr` and `handle`) and map it into the requested VMA. Device drivers could use this function to make a buffer available to user space, and would be able to discard their existing `nopage()` methods. The new interface would thus simplify things, though it does still leave a reference counting problem on the driver side of things: freeing the DMA buffer before user space has unmapped it would be a big mistake.
