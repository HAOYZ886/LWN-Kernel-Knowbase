---
title: Noncoherent DMA mappings
url: https://lwn.net/Articles/855328/
date: "May 7, 2021"
category: Direct memory access
author: "By Jonathan Corbet May 7, 2021"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 7, 2021

While it is sometimes possible to perform I/O by moving data through the CPU, the only way to get the required level of performance is usually for devices to move data directly to and from memory. Direct memory access (DMA) I/O has been well supported in the Linux kernel since the early days, but there are always ways in which that support can be improved, especially when hardware adds some challenges of its own. The somewhat confusingly named "non-contiguous" DMA API that was added for 5.13 shows the kinds of things that have to be done to get the best performance on current systems. 

DMA, of course, presents a number of interesting race conditions that can arise in the absence of an agreement between the CPU and the device over who controls a range of memory at any given time. But there is another problem that comes up as well. CPUs aggressively cache memory contents to avoid the considerable expense of actually going to memory for every reference. But if a CPU has cached data that is subsequently overwritten by DMA, the CPU could end up reading incorrect data from the cache, resulting in data corruption. Similarly, if the cache contains data written by the CPU that has not yet made it to memory, that data really needs to be flushed out before the device accesses that memory or bad things are likely to happen. 

The x86 architecture makes life relatively easy (in this regard, at least) for kernel developers by providing cache snooping; CPU caches will be invalidated if a device is seen to be writing to a range of memory, for example. This "cache-coherent" behavior means that developers need not worry about cache contents corrupting their data. Other architectures are not so forgiving. The Arm architecture, among others, will happily retain cache contents that no longer match the memory they are allegedly caching. On such systems, developers must take care to manage the cache properly as control of a DMA buffer is passed between the device and the CPU. 

There are a number of ways to handle this task, but life gets harder if a DMA buffer also requires extensive access by either the kernel or user space. One approach that is taken at times is to make that range of memory uncached. A nonexistent cache cannot corrupt data, but it can make it clear why caches exist in the first place; accessing uncached memory can be extremely slow. If at all possible, it is better to avoid the uncached mode. 

The new API is a way to do that for some sorts of devices. A driver can allocate a DMA buffer using: 

```
struct sg_table *dma_alloc_noncontiguous(struct device *dev, size_t size,
    					     enum dma_data_direction direction,
    					     gfp_t gfp, unsigned long attrs);
```

This function will attempt to allocate `size` bytes of memory for DMA by `dev` in the given [`direction`](<https://elixir.bootlin.com/linux/v5.12.1/source/include/linux/dma-direction.h#L5>) using the given `gfp` flags. That buffer may not be physically contiguous in system memory, but the returned scatter/gather table will be set up with a single, contiguous range for the DMA device. An I/O memory-management unit (IOMMU) is clearly required for the system to be able to arrange that; it's an important feature, though, since some devices cannot do scatter/gather I/O without IOMMU assistance. The only accepted value for `attrs` is `DMA_ATTR_ALLOC_SINGLE_PAGES`, which is a hint that it's not worthwhile for the DMA-mapping code to try to use huge pages for this buffer. 

This buffer may well not be cache-coherent. As with other noncoherent mappings, cache management must be done by hand. So, for example, a call to [`dma_sync_sgtable_for_device()`](<https://elixir.bootlin.com/linux/v5.12.1/source/include/linux/dma-mapping.h#L375>) must be done before handing the memory over to the device for I/O; it will make sure that any dirty cache lines will be flushed out to the memory, among other things. To take control back from the device, [`dma_sync_sgtable_for_cpu()`](<https://elixir.bootlin.com/linux/v5.12.1/source/include/linux/dma-mapping.h#L356>) must be called. 

The buffer can be freed with: 

```
void dma_free_noncontiguous(struct device *dev, size_t size,
        				struct sg_table *sgtable,
    				enum dma_data_direction dir);
```

The parameters must match those used when the buffer was allocated. 

This buffer is not directly accessible by the CPU when returned. If the kernel needs a mapping into kernel space, that can be managed with: 

```
void *dma_vmap_noncontiguous(struct device *dev, size_t size,
        				 struct sg_table *sgtable);
        void dma_vunmap_noncontiguous(struct device *dev, void *vaddr);
```

The existence of a kernel mapping does not make cache-coherency issues go away, though. If the kernel may have written to this buffer, `flush_kernel_vmap_range()` must be called to ensure any cached data makes it to memory before handing that memory to a device. Similarly, `invalidate_kernel_vmap_range()` must be called to remove any cached data for memory that may have been written by the device. 

Finally, it is possible to map the buffer into user space with a call to: 

```
int dma_mmap_noncontiguous(struct device *dev, struct vm_area_struct *vma,
    			       size_t size, struct sg_table *sgt);
```

This will normally be done in response to an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call by the application, which can `munmap()` the memory when it's no longer needed. Needless to say, if user space accesses this buffer when it is in the device's hands, the results may be less than optimal. In cases where the ownership of the buffer is managed explicitly in user space (such as with the Video4Linux2 API, for example), access at the wrong time should not be a problem. 

Also merged in 5.13 was [a patch to the uvcvideo driver](<https://git.kernel.org/linus/20e1dbf2bbe2>) to switch it from using coherent mappings to the new API. According to the changelog, this change can, on non-cache-coherent systems, improve performance by a factor of 20, which seems worth the effort. Chances are that other drivers will make the switch at some point. It's the kind of change that is not immediately evident to users, but which makes the system perform much better in the end.
