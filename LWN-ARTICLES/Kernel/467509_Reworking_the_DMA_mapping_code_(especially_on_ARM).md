---
title: Reworking the DMA mapping code (especially on ARM)
url: https://lwn.net/Articles/467509/
date: "November 16, 2011"
category: "Architectures-Arm; Direct memory access"
author: "By Jonathan Corbet November 16, 2011"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
November 16, 2011

Direct memory access (DMA) I/O is a simple-sounding concept: devices are able to access memory directly and transfer data without involving the CPU. In practice, of course, it turns into a complex problem when confronted with the real world and its strange architectural differences, problematic devices, and varying I/O needs. The DMA mapping API was created as a way to minimize the amount of DMA-related complexity that drivers have to deal with, a goal it has achieved fairly well. Changing needs, and increasing hardware complexity are driving some changes in this area, though, with the side benefit that the ARM architecture should get a nice cleanup as well. 

As is the case in many areas, the ARM architecture has its own implementation of the DMA API, despite the fact that there is quite a bit of architecture-independent code available to be used. The usual reasons apply here: a combination of developers only working in the ARM tree and peculiarities specific to that architecture. It is a pattern that has been seen in many other places; it is certainly not specific to ARM. 

One of the first things done by Marek Szyprowski's [ARM DMA redesign patch set](<https://lwn.net/Articles/465047/>) is to hook ARM into the common DMA mapping framework. That enables the deletion of a certain amount of duplicated code and its replacement with common code. Among other things, this work simplifies the handling of differences within the ARM architecture itself. Through the use of the common `struct dma_map_ops`, an architecture can provide a set of mapping operations specific to a given situation - different devices can have different DMA operations, for example. 

But there is more to ARM's DMA implementation than the common interface; ARM's API has special functions like: 

```
void *dma_alloc_writecombine(struct device *dev, size_t len, 
    				 dma_addr_t *dma_addr, gfp_t flags);
```

This function allocates a DMA buffer with "write combining" attributes, meaning that data written to that memory (by the CPU) may be delayed by the memory hardware and flushed out in batches. Use of write-combining memory can yield significant performance improvements for some device types, but this memory clearly has to be handled carefully so that deferred writes don't get mixed up with accesses by the device. A number of drivers use this function, but only one other architecture (avr32) provides it. 

ARM also has special functions for mapping DMA buffers into user space: 

```
int dma_mmap_coherent(struct device *dev, struct vm_area_struct *vma,
    			  void *cpu_addr, dma_addr_t dma_addr, size_t len);
```

On most architectures, memory-mapping a coherent buffer requires no special handling, so the generic DMA code does not provide any special support for this operation; only one other architecture (PowerPC) has felt the need to add this function. 

Clearly, bringing the ARM DMA API into line with common code will require some way of handling these special functions. The fact that, for each of the above functions, one other architecture has added an implementation indicates that ARM, as strange as it is, is not alone in needing an expanded API. So the logical thing to do is to move support for these functions into the common DMA core implementation. 

That could be done by adding new `alloc_writecombine()` and `mmap_coherent()` functions (and, yes, `mmap_writecombine()` too) to `struct dma_map_ops`. As the number of combinations of operations and memory attributes grows, though, the size of that structure will grow as well. Marek decided to take a different approach; his patch removes the existing `alloc_coherent()` and `free_coherent()` members, replacing them with: 

```
void* (*alloc)(struct device *dev, size_t size, dma_addr_t *dma_handle, 
    		   gfp_t gfp, struct dma_attrs *attrs);
        void (*free)(struct device *dev, size_t size, void *vaddr,
    		 dma_addr_t dma_handle, struct dma_attrs *attrs);
        int (*mmap)(struct device *dev, struct vm_area_struct *vma, void *cpu_addr,
    		dma_addr_t dma_addr, size_t size, struct dma_attrs *attrs);
```

As it happens, `struct dma_attrs` already exists in current kernels. It is not heavily used, though; there are currently only two attributes defined (described in [Documentation/DMA-attributes.txt](<https://lwn.net/Articles/467516/>)) that seem to only be implemented in the ia64 and PowerPC/Cell architectures. Only one of them (`DMA_ATTR_WRITE_BARRIER`) seems to actually be used, and in only one place (the InfiniBand code). But the mechanism already exists, so adding more attributes seems like a better approach than adding a new way to express things like "write combining." Marek's patch adds the convention that a null `attrs` pointer means "coherent," then adds attributes for noncoherent and write-combining mappings. The various allocation functions can then be replaced with: 

```
void *dma_alloc_attrs(struct device *dev, size_t size, 
    			  dma_addr_t *dma_handle, gfp_t flag, 
    			  struct dma_attrs *attrs);
```

This function can be used to request a mapping with any set of attributes that the underlying platform may support; similar functions exist for freeing and memory-mapping DMA buffers. Marek's patch does not extend this functionality into other architectures - even those that have added functions similar to those used by ARM - but that seems like an obvious next step. 

Once that is done, Marek can get to what was perhaps his real goal: adding support for per-device I/O memory management units (IOMMUs) to the ARM DMA API. Some hardware has a separate IOMMU built into it that cannot be used for other devices, so the IOMMU cannot be made available to the system as a whole. But it is possible to attach a device-specific `dma_map_ops` structure to such devices that would cause the DMA API to use the IOMMU without the device driver even needing to know about it. And that, of course, leads to simpler and more reliable code. 

Prior to this work, IOMMU awareness had been built into specific drivers directly. But that caused opposition at review time; drivers written in that way cannot really be merged into the mainline. When he talked about this work at LinuxCon Prague, Marek passed on a few lessons that he had learned from the experience. The first of those is that one should always use existing APIs whenever possible. Every developer thinks they can do something better; that may or may not be true, but using the common code works out better in the long run. But, he said, developers should not be afraid of extending core interfaces when the need arises. That is how problems get solved and how the core gets better. The final lesson was "expect it to take some time" when one has to solve problems of this nature. 

On the subject of time: it is not clear when this work might make it into the mainline. It has not yet really been submitted for inclusion; the current patches have some obvious work that needs to be done before they are ready. But Marek, after a number of tries, appears to have gotten past the serious technical objections and is now working on getting the details right. So, while one should follow his advice and expect it to take some time, the value of "some time" should be approaching a reasonably small number.
