---
title: "Dancing the DMA two-step"
url: https://lwn.net/Articles/997563/
date: "November 14, 2024"
category: "Device drivers-Support APIs; Direct memory access"
author: "By Jonathan Corbet November 14, 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
November 14, 2024

Direct memory access (DMA) I/O is simple in concept: a peripheral device moves data directly to or from memory while the CPU is busy doing other things. As is so often the case, DMA is rather more complicated in practice, and the kernel has developed a complicated internal API to support it. It turns out that the DMA API, as it exists now, can affect the performance of some high-bandwidth devices. In an effort to address that problem, Leon Romanovsky is making the API even more complex with [this patch series](<https://lwn.net/ml/all/cover.1731244445.git.leon@kernel.org>) adding a new two-step mapping API. 

#### DMA challenges

In the early days, a device driver would initiate a DMA operation by passing the physical address of a memory buffer to the device and telling it to go. There are a number of reasons why things cannot be so simple on current systems, though, including: 

  * The device may not be able to reach the buffer. ISA devices were limited to 24-bit DMA addresses, for example, so any memory located above that range was inaccessible to those devices. More recently, many devices were still limited to 32-bit addresses, though hopefully that situation has improved over time. If a buffer is out of a device's reach, it must be copied into reachable memory (a practice known as "bounce buffering") before setting up the I/O operation. 
  * The combination of memory caching in the CPU and DMA can lead to inconsistent views of the data held in memory — the device cannot see data that exists only in the cache, for example. If not properly managed, cache consistency (or the lack thereof) can lead to data corruption, which is usually deemed to be a bad thing. 
  * The buffer involved in a transfer may be scattered throughout physical memory; for larger transfers, it is almost guaranteed to be. The kernel's DMA layer manages the scatter/gather lists ("scatterlists") needed to describe these operations. 
  * Modern systems often do not give devices direct access to the physical memory space; instead, that access is managed through an I/O memory-management unit (IOMMU), which creates an independent address space for peripheral devices. Any DMA operation requires setting up a mapping within the IOMMU to allow the device to access the buffer. An IOMMU can make a physically scattered buffer look contiguous to a device. It may also be able to prevent the device from accessing memory outside of the buffer; this capability is necessary to safely allow virtual machines to directly access I/O devices. 
  * DMA operations between two peripheral devices (without involving main memory at all) — [P2PDMA](<https://lwn.net/Articles/767281/>) — add a whole new level of complexity. 

To top it all off, a device driver usually cannot be written with a knowledge of the organization of every system on which it will run, so it must be able to adapt to the DMA-mapping requirements it finds. 

All of this calls out for a kernel layer to abstract the DMA-mapping task and present a uniform interface to device drivers. The kernel [has such a layer](<https://docs.kernel.org/core-api/dma-api.html>), which has been present in something close to its current form for some years. At the core of this layer is the scatterlist API. As Romanovsky notes in the patch cover letter, though, this API has been showing signs of strain for some time. 

Scatterlists are used heavily in the DMA API, but they are fundamentally based on the kernel's `page` structure, which describes a single page of memory. That makes scatterlists unable to deal with larger groupings of pages (folios) without splitting them into individual pages. Being based on `struct page` also complicates P2PDMA; since only device memory is involved for those operations, there are no `page` structures to use. Increasingly, I/O operations are already represented in the kernel in a different form (an array of `bio` structures for a block operation, for example), reformatting that information into a scatterlist is mostly unnecessary overhead. So there has been interest in improving or replacing scatterlists for some time; see [the phyr discussion from 2023](<https://lwn.net/Articles/931943/>) for example. So far, though, scatterlists have proved resistant to these efforts. 

#### Splitting things up

Romanovsky has set out to create a DMA API that will address many of the complaints about scatterlists while improving performance. The core idea, he says is to ""instead split up the DMA API to allow callers to bring their own data structure"". The split, in this case, is between the allocation of an I/O virtual address (IOVA) space for an operation and the mapping of memory into that space. This new API is intended to be a supplemental option on high-end systems with IOMMUs; it will not replace the existing DMA API. 

The first step when using this new API is to allocate a range of IOVA space to be used with the upcoming transfer(s): 

```
bool dma_iova_try_alloc(struct device *dev, struct dma_iova_state *state,
    			    phys_addr_t phys, size_t size);
```

This function will attempt to allocate a `size`-byte IOVA range for use by the given device (`dev`). The `phys` argument only indicates the necessary alignment for this range; for devices that only require page alignment, passing zero will work. The `state` structure must be provided by the caller, but will be completely initialized by this call. 

If the allocation attempt is successful, this function will return `true` and the physical address of the range (as seen by the device) will be stored in `state.addr`. Otherwise, the return value will be `false`, and the older DMA API must be used instead. Thus, the new API does not enable the removal of scatterlist support from any drivers; it just provides a higher-performance alternative on systems where it is supported. 

If the allocation is successful, the result is an allocated range of IOVA space that does not yet map to anything. The driver can map ranges of memory into this IOVA area with: 

```
int dma_iova_link(struct device *dev, struct dma_iova_state *state,
    		      phys_addr_t phys, size_t offset, size_t size,
    		      enum dma_data_direction dir, unsigned long attrs);
```

Here `dev` is the device that will be performing the I/O (the same one that was used to allocate the IOVA space), `state` is the state structure used to allocate the address range, `phys` is the physical address of the memory range to map, `offset` is the offset into the IOVA range where this memory should be mapped, `size` is the size of the range to be mapped, `dir` describes the I/O direction (whether data is moving to or from the device), and `attrs` holds the optional [attributes](<https://elixir.bootlin.com/linux/v6.11.6/source/include/linux/dma-mapping.h#L15>) that can modify the mapping. The return value will be zero (for success) or a negative error code. 

Once all of the memory has been mapped, the driver should make a call to: 

```
int dma_iova_sync(struct device *dev, struct dma_iova_state *state,
     		      size_t offset, size_t size);
```

This call will synchronize the I/O translation lookaside buffer (an expensive operation that should only be done once, after the mapping is complete) for the indicated range of the IOVA area. Then the I/O operation can be initiated. 

Afterward, portions of the IOVA range can be unmapped with: 

```
void dma_iova_unlink(struct device *dev, struct dma_iova_state *state,
    			 size_t offset, size_t size, enum dma_data_direction dir,
    			 unsigned long attrs);
```

Once all the mappings have been unlinked, the IOVA can be freed with: 

```
void dma_iova_free(struct device *dev, struct dma_iova_state *state);
```

Alternatively, a call to: 

```
void dma_iova_destroy(struct device *dev, struct dma_iova_state *state,
    			  size_t mapped_len, enum dma_data_direction dir,
    			  unsigned long attrs);
```

will unmap the entire range (up to `mapped_len`), then free the IOVA allocation. 

In summary, Romanovsky is proposing an API that can be used to map a scattered set of buffers into a single, contiguous IOVA range. There is no need to create a separate scatterlist data structure to represent this operation, and there is no need to use `page` structures to refer to the memory. 

#### Current state

This API has been through a few revisions at this point, and some developers, at least, are happy with it. While the new API provides improved performance for some use cases, Jens Axboe has [observed](<https://lwn.net/ml/all/3144b6e7-5c80-46d2-8ddc-a71af3c23072@kernel.dk/>) performance regressions within the block layer that are not yet understood. For now, Romanovsky has [removed](<https://lwn.net/ml/all/20241031090530.GC7473@unreal/>) some of the block-layer changes that he deems to be the most likely source of the problem. 

Robin Murphy has, instead, [questioned](<https://lwn.net/ml/all/3567312e-5942-4037-93dc-587f25f0778c@arm.com/>) one of the core assumptions of this API: that there is value in mapping scatter/gather operations into a contiguous IOVA range: 

> TBH I doubt there are many actual scatter-gather-capable devices with significant enough limitations to meaningfully benefit from DMA segment combining these days - I've often thought that by now it might be a good idea to turn that behaviour off by default and add an attribute for callers to explicitly request it. 

Christoph Hellwig [responded](<https://lwn.net/ml/all/20241104095831.GA28751@lst.de/>) that devices often perform better with a contiguous IOVA range, even if they are able to handle a severely fragmented one. Jason Gunthorpe [agreed](<https://lwn.net/ml/all/20241105195357.GI35848@ziepe.ca/>), saying that RDMA operations see ""big wins"" when the IOVA range is contiguous. So it does appear that there is a need for this capability. 

The patch set seems to have reasonably broad support, and the rate of change appears to be slowing. There are, of course, possible improvements to the API that could be considered; Gunthorpe mentioned better control over alignment in the above-linked message, for example, but those can come later. Romanovsky has [asked](<https://lwn.net/ml/all/20241114133011.GA606631@unreal>) that it be merged for 6.13 so that drivers can easily start to use it. While there are no guarantees at this point (and some [resistance to the idea](<https://lwn.net/ml/all/20241114163622.GA3121@lst.de>)), it seems possible that the next kernel will include a new, high-performance DMA API.
