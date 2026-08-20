---
title: Generic DMA pools
url: https://lwn.net/Articles/69402/
date: "February 3, 2004"
category: Device drivers; Direct memory access
author: 
---

Device driver authors sometimes find that they have to perform DMA operations on very small pieces of memory. It is tempting to just perform this sort of DMA (often just a few bytes) directly into or out of a kernel data structure. The problem with this approach is that caching issues can arise; memory adjacent to the region being read or written by the device can end up with the wrong values. Needless to say, this sort of memory corruption is not good for long-term system stability. 

This problem can be avoided through the use of "PCI pools." A PCI pool is simply a source of small pieces of memory which are suitable for DMA operations. A driver which makes use of a PCI pool for its small DMA needs will not have memory corruption issues. 

There is only one problem with PCI pools: not all devices are attached to a PCI bus. With the intent of making the PCI pool functionality available to a wider class of devices, Deepak Saxena has posted [a set of patches](<https://lwn.net/Articles/69345/>) implementing a new "DMA pool" abstraction. The new interface is strikingly similar to the old one - to the point that the old `pci_pool_` functions can be emulated with simple macros. As a result, drivers using the old PCI functions will continue to work without changes. 

In the new scheme, DMA pools are allocated and destroyed with: 

```
struct dma_pool *dma_pool_create(const char *name, struct device *dev,
                                         size_t size, size_t align,
    				     size_t allocation);
        void dma_pool_destroy(struct dma_pool *pool);
```

Parameters for the creation of the pool include its name, the device which will be using the pool, the size of blocks to be allocated from the pool, and the required alignment. Optionally, the `allocation` parameter can be used to keep pool memory from crossing a specific memory size barrier; if `allocation` is 4096, for example, no pool allocation will cross a 4K page boundary. The main difference from the old `pci_pool_create()` function is the use of a `device` structure rather than a `pci_dev` structure. 

The allocation and deallocation functions are: 

```
void *dma_pool_alloc(struct dma_pool *pool, int mem_flags,
                             dma_addr_t *handle);
        void dma_pool_free(struct dma_pool *pool, void *vaddr, 
                           dma_addr_t handle);
```

Internally, the new pool functions bear a strong resemblance to the old ones - with the obvious exception that the memory for the pools is now allocated using the generic DMA functions. 

This patch has been received well; chances are it will appear in a kernel sometime after 2.6.2 comes out.
