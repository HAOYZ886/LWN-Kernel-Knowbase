---
title: Scatterlist chaining
url: https://lwn.net/Articles/234617/
date: "May 16, 2007"
category: Scattergather chaining
author: 
---

High-performance I/O generally involves the use of direct memory access (DMA) operations. With DMA, the I/O device transfers data directly to and from main memory without the intervention of the CPU. In the simplest form of DMA, the controller is handed a pointer to a region of memory, given the length, and told to do its thing. The processor can then forget about the operation until the device signals that the work is done. 

This simple view has a drawback, however, in that it assumes that the data to be transferred is stored contiguously in memory. When the I/O buffer is in kernel space, the kernel can often arrange for it to be physically contiguous - though that gets harder as the size of the buffers gets larger. If the buffer is in user space, it is guaranteed to be scattered around physical memory. So it would be nice if DMA operations could work with buffers which are split into a number of distinct pieces. 

In fact, with any reasonably capable peripheral device, buffers can be split this way. The term for operations on such buffers is "scatter/gather I/O"; scatter/gather has been well supported under Linux for some time. The DMA chapter of [_Linux Device Drivers_](<https://lwn.net/Kernel/LDD3/>) covers scatter/gather in a fair amount of detail. In short, a driver starts by filling in an array of `scatterlist` structures, which (on the i386 architecture) look like: 

```
struct scatterlist {
            struct page	*page;
        	unsigned int	offset;
        	dma_addr_t	dma_address;
        	unsigned int	length;
        };
```

For each segment, the `page` pointer tells where the segment is to be found in memory, `offset` tells where the data begins within the page, and `length` is the length of the segment. Once the list has been filled in, the driver calls: 

```
int dma_map_sg(struct device *dev, struct scatterlist *sg, int nents,
                       enum dma_data_direction direction);
```

This operation, at a minimum, fills in the `dma_address` field of each scatterlist entry with an address which can be given to the peripheral. It might do more, though: physically contiguous pages may be coalesced into a single `scatterlist` entry, or the system's I/O memory management unit might be programmed to make parts (or all) of the list virtually contiguous from the device's point of view. All of this - including the exact form of `struct scatterlist` \- is architecture dependent, but the scatter/gather interface is set up so that drivers need not worry about architecture details. 

Recently, a particular constraint in the scatter/gather interface has turned up. For various reasons, scatterlists must fit within a single page; that restriction puts an upper limit on the number of segments which may be represented. On the i386 architecture, with high memory enabled, `struct scatterlist` requires 20 bytes, which limits a scatterlist to 204 entries. If each scatterlist entry points to a full page, the maximum size for a DMA transfer will be about 800KB. On an x86-64 system, the situation is worse: the structure uses 40 bytes, cutting the maximum length in half. 

There are situations where larger I/O operations are desirable. The block I/O subsystem is one of them, but there are certainly others: high-resolution video capture devices are an example. The limitation on scatterlist length is one of the factors motivating developers who are working on [large block size support](<http://lwn.net/Articles/232757/>). By increasing the effective page size, they are able to increase the maximum I/O operation size. 

Increasing the page size is not the only feasible approach, though; another is simply to make scatterlists longer. Multi-page contiguous scatterlists are not really in the cards, but chaining single-page scatterlists can be done. Jens Axboe has been working on that approach; his [scatterlist chaining patch](<http://lwn.net/Articles/234605/>) is on its sixth revision as of this writing. 

Chaining is done by overloading the `page` pointer in the last scatterlist entry in a page. The least significant bit is set to indicate that the entry is, in fact, a chain link rather than another segment to transfer. The change is _almost_ transparent to drivers. In current kernels, the code which iterates through a scatterlist usually looks something like this: 

```
struct scatterlist *sg = &the_scatterlist[0];
    
        for (i = 0; i < nentries; i++) {
    	program_io_operation(sg);
    	sg++;
        }
```

When chaining is being used, simply incrementing through the array no longer works. So Jens has added a simple `sg_next()` macro to follow the the chain links when necessary. So the `sg++` line above turns into something like: 

```
sg = sg_next(sg);
```

Since a driver change is required, chained scatterlists should not be used unless one knows for sure that the driver is prepared for them. The patch from Jens fixes up a number of drivers, especially in the block subsystem. Even so, the maximum I/O size must be raised explicitly by the administrator (via a sysfs file) before chaining will be turned on. Once it's enabled, however, multi-megabyte I/O operations become possible. No intrusive memory management changes required.
