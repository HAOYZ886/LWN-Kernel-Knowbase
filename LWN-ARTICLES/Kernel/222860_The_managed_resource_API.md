---
title: The managed resource API
url: https://lwn.net/Articles/222860/
date: "February 20, 2007"
category: Resources
author: 
---

The [device resource management patch](<http://lwn.net/Articles/215996/>) was discussed here in January. That patch has now been merged for the 2.6.21 kernel. Since the API is now set - at least, as firmly as any in-kernel API is - it seems like a good time for a closer look at this new interface. 

The core idea behind the resource management interface is that remembering to free allocated resources is hard. It appears to be especially hard for driver writers who, justly or not, have a reputation for adding more than their fair share of bugs to the kernel. And even the best driver writers can run into trouble in situations where device probing fails halfway through; the recovery paths may be there in the code, but they tend not to be well tested. The result of all this is a fair number of resource leaks in driver code. 

To address this problem, Tejun Heo created a new set of resource allocation functions which track allocations made by the driver. These allocations are associated with the `device` structure; when the driver detaches from the device, any left-over allocations are cleaned up. The resource management interface is thus similar to the [`talloc()` API](<http://samba.org/ftp/unpacked/samba4/source/lib/talloc/talloc_guide.txt>) used by the Samba hackers, but it is adapted to the kernel environment and covers more than just memory allocations. 

Starting with memory allocations, though, the new API is: 

```
void *devm_kzalloc(struct device *dev, size_t size, gfp_t gfp);
        void devm_kfree(struct device *dev, void *p);
```

In a pattern we'll see repeated below, the new functions are similar to `kzalloc()` and `kfree()` except for the new names and the addition of the `dev` argument. That argument is necessary for the resource management code to know when the memory can be freed. If any memory allocations are still outstanding when the associated device is removed, they will all be freed at that time. 

Note that there is no managed equivalent to `kalloc()`; if driver writers cannot be trusted to free memory, it seems, they cannot be trusted to initialize it either. There are also no managed versions of the page-level or slab allocation functions. 

Managed versions of a subset of the DMA allocation functions have been provided: 

```
void *dmam_alloc_coherent(struct device *dev, size_t size,
    			      dma_addr_t *dma_handle, gfp_t gfp);
        void dmam_free_coherent(struct device *dev, size_t size, void *vaddr,
    			    dma_addr_t dma_handle);
        void *dmam_alloc_noncoherent(struct device *dev, size_t size,
    			         dma_addr_t *dma_handle, gfp_t gfp);
        void dmam_free_noncoherent(struct device *dev, size_t size, void *vaddr,
    			       dma_addr_t dma_handle);
        int dmam_declare_coherent_memory(struct device *dev, dma_addr_t bus_addr,
    				     dma_addr_t device_addr, size_t size, 
    				     int flags);
        void dmam_release_declared_memory(struct device *dev);
        struct dma_pool *dmam_pool_create(const char *name, struct device *dev,
    				      size_t size, size_t align,
    				      size_t allocation);
        void dmam_pool_destroy(struct dma_pool *pool);
```

All of these functions have the same arguments and functionality as their `dma_`* equivalents, but they will clean up the DMA areas on device shutdown. One still has to hope that the driver has ensured that no DMA remains active on those areas, or unpleasant things could happen. 

There is a managed version of `pci_enable_device()`: 

```
int pcim_enable_device(struct pci_dev *pdev);
```

There is no `pcim_disable_device()`, however; code should just use `pci_disable_device()` as usual. A new function: 

```
void pcim_pin_device(struct pci_dev *pdev);
```

will cause the given `pdev` to be left enabled even after the driver detaches from it. 

The patch makes the allocation of I/O memory regions with `pci_request_region()` managed by default - there is no `pcim_` version of that interface. The higher-level allocation and mapping interfaces do have managed versions: 

```
void __iomem *pcim_iomap(struct pci_dev *pdev, int bar, 
                                 unsigned long maxlen);
        void pcim_iounmap(struct pci_dev *pdev, void __iomem *addr);
```

For the allocation of interrupts, the managed API is: 

```
int devm_request_irq(struct device *dev, unsigned int irq,
    		         irq_handler_t handler, unsigned long irqflags,
    		     	 const char *devname, void *dev_id);
        void devm_free_irq(struct device *dev, unsigned int irq, void *dev_id);
```

For these functions, the addition of a `struct device` argument was required. 

There is a new set of functions for the mapping of of I/O ports and memory: 

```
void __iomem *devm_ioport_map(struct device *dev, unsigned long port,
    			          unsigned int nr);
        void devm_ioport_unmap(struct device *dev, void __iomem *addr);
        void __iomem *devm_ioremap(struct device *dev, unsigned long offset,
    			       unsigned long size);
        void __iomem *devm_ioremap_nocache(struct device *dev, 
                                           unsigned long offset,
    				       unsigned long size);
        void devm_iounmap(struct device *dev, void __iomem *addr);
```

Once again, these functions required the addition of a `struct device` argument for the managed form. 

Finally, for those using the low-level resource allocation functions, the managed versions are: 

```
struct resource *devm_request_region(struct device *dev,
    				         resource_size_t start,
    					 resource_size_t n, 
    					 const char *name);
        void devm_release_region(resource_size_t start, resource_size_t n);
        struct resource *devm_request_mem_region(struct device *dev,
    				             resource_size_t start,
    					     resource_size_t n, 
    					     const char *name);
        void devm_release_mem_region(resource_size_t start, resource_size_t n);
```

The resource management layer includes a "group" mechanism, accessed via these functions: 

```
void *devres_open_group(struct device *dev, void *id, gfp_t gfp);
        void devres_close_group(struct device *dev, void *id);
        void devres_remove_group(struct device *dev, void *id);
        int devres_release_group(struct device *dev, void *id);
```

A group can be thought of as a marker in the list of allocations associated with a given device. Groups are created with `devres_open_group()`, which can be passed an `id` value to identify the group or `NULL` to have the ID generated on the fly; either way, the resulting group ID is returned. A call to `devres_close_group()` marks the end of a given group. Calling `devres_remove_group()` causes the system to forget about the given group, but does nothing with the resources allocated within the group. To remove the group and immediately free all resources allocated within that group, `devres_release_group()` should be used. 

The group functions seem to be primarily aimed at mid-level code - the bus layers, for example. When bus code tries to attach a driver to a device, for example, it can open a group; should the driver attach fail, the group can be used to free up any resources allocated by the driver. 

There are not many users of this new API in the kernel now. That may change over time as driver writers become aware of these functions, and, perhaps, as the list of managed allocation types grows. The reward for switching over to managed allocations should be more robust and simpler code as current failure and cleanup paths are removed.
