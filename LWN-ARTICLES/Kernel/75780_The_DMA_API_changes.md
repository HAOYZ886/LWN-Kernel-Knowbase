---
title: The DMA API changes
url: https://lwn.net/Articles/75780/
date: "March 15, 2004"
category: Direct memory access
author: 
---

The 2.6 kernel is a stable series which, in theory, should be dedicated to the fixing of bugs rather than changing APIs. Anybody who risks thinking that things have become too stable, however, need only look at [this massive patch](<https://lwn.net/Articles/75781/>) from David Miller, which changes the DMA API and touches a full 100 files. This patch had done a little time in the -mm tree, but had never really been discussed on the mailing lists before its inclusion. 

The change is in the "synchronization" calls that the DMA layer provides for streaming mappings. A streaming mapping is a short-lived structure set up to support one or more direct memory access operations; depending on the architecture, setting up a streaming mapping can involve creating bounce buffers, programming I/O memory management unit (IOMMU) registers, flushing processor caches, and more. These mappings have strict rules about the "ownership" of the buffer; when a streaming mapping is created, it is owned by the device, and the processor cannot touch it. If a device driver ignores that rule, it risks corrupting data in a number of ways. 

It is sometimes necessary, however, to allow the processor to access a mapped streaming DMA buffer. To that end, the DMA layer has long provided a set of functions (like `dma_sync_single()` and `pci_sync_single()`) which transfer ownership of the buffer to the CPU. What has always been lacking, however, is a way to transfer ownership back to the device. To fill in that gap, the various synchronization functions have been split in two; instead of `dma_sync_single()` a driver must now call one or both of: 

```
dma_sync_single_for_cpu(struct device *dev, 
                                dma_addr_t dma_handle, 
    			    size_t size,
    			    enum dma_data_direction direction);
    
        dma_sync_single_for_device(struct device *dev, 
                                   dma_addr_t dma_handle, 
    			       size_t size,
    			       enum dma_data_direction direction);
```

`dma_sync_single_for_cpu()` gives ownership of the DMA buffer back to the processor. After that call, driver code can read or modify the buffer, but the device should not touch it. A call to `dma_sync_single_for_device()` is required to allow the device to access the buffer again. The other synchronization functions (for scatter/gather and DAC mappings) have been changed as well. 

As might be expected from a change like this, the result was a lot of broken drivers. The patch fixes the in-tree users of the discontinued DMA functions. Out-of-tree and binary-only drivers, however, will have to be fixed separately.
