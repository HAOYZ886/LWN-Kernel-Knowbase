---
title: "DMA issues, part 2"
url: https://lwn.net/Articles/91870/
date: "June 30, 2004"
category: Direct memory access; dma declare coherent memory; swiotlb
author: 
---

[Last week's Kernel Page](<https://lwn.net/Articles/89961/>) looked at various DMA-related issues. One of those was the ability to make use of memory located on I/O controllers for DMA operations. That work has taken a step forward with [this proposal](<https://lwn.net/Articles/91664/>) from James Bottomley, which adds a new function to the DMA API: 

```
int dma_declare_coherent_memory(struct device *dev, 
                                        dma_addr_t bus_addr,
                                        dma_addr_t device_addr, 
                                        size_t size, int flags);
```

This function tells the DMA code about a chunk of memory available on the device represented by `dev`. The memory is `size` bytes long; it is located at `bus_addr` from the bus's point of view, and `device_addr` from the device's perspective. The `flags` argument describes how the memory is to be used: whether it should be mapped into the kernel's address space, whether children of the device can use it, and whether it should be the only memory used by the device(s) for DMA. 

The actual patch implementing this API is still in the works. As of this writing, there have been no real comments on it. 

Meanwhile, a different DMA issue has been raised by the folks at nVidia, who are trying to make their hardware work better on Intel's em64t (AMD64 clone) architecture. It is, it turns out, difficult to reliably use DMA on devices which cannot handle 64-bit addresses. 

Memory on (non-NUMA) Linux systems has traditionally been divided into three zones. `ZONE_DMA` is the bottom 16MB; it is the only memory which is accessible to ancient ISA peripherals and, perhaps, a few old PCI cards which are simply a repackaging of ISA chipsets. `ZONE_NORMAL` is all of the memory, outside of ZONE_DMA, which is directly accessible to the kernel. On a typical 32-bit Linux system, `ZONE_NORMAL` extends up to just under the first 1GB of physical memory. Finally, `ZONE_HIGHMEM` is the "high memory" zone - the area which is not directly accessible to the kernel. 

This layout works reasonably well for DMA allocations on 32-bit systems. Truly limited peripherals use memory taken from `ZONE_DMA`; most of the rest work with `ZONE_NORMAL` memory. In the 64-bit world, however, things are a little different. There is no need for high memory on such systems, so `ZONE_HIGHMEM` simply does not exist, and `ZONE_NORMAL` contains everything above `ZONE_DMA`. Having (almost) all of main memory contained within `ZONE_NORMAL` simplifies a lot of things. 

Kernel memory allocations specify (implicitly or explicitly) the zone from which the memory is to be obtained. On 32-bit systems, the DMA code can simply specify a zone which matches the capabilities of the device and get the memory it needs. On 64-bit systems, however, the memory zones no longer align with the limitations of particular devices. So there is no way for the DMA layer to request memory fitting its needs. The only exception is `ZONE_DMA`, which is far more restrictive than necessary. 

On some architectures - notably AMD's x86_64 - an I/O memory management unit (IOMMU) is provided. This unit remaps addresses between the peripheral bus and main memory; it can make any region of physical memory appear to exist in an area accessible by the device. Systems equipped with an IOMMU thus have no problems allocating DMA memory - any memory will do. Unfortunately, when Intel created its variant of the x86_64 architecture, it decided to leave the IOMMU out. So devices running on "Intel inside" systems work directly with physical memory addresses, and, as a result, the more limited devices out there cannot access all of physical memory. And, as we have seen, the kernel has trouble allocating memory which meets their special needs. 

One solution to this problem could be the creation of a new zone, `ZONE_BIGDMA`, say, which would represent memory reachable with 32-bit addresses. Nobody much likes this approach, however; it involves making core memory management changes to deal with the shortcomings of a few processors. Balancing memory use between zones is a perennial memory management headache, and adding more zones can only make things worse. There is one other problem as well: some devices have strange DMA limitations (a maximum of 29 bits, for example); creating a zone which would work for all of them would not be easy. 

The Itanium architecture took a different approach, known as the "software I/O translation buffer" or "swiotlb." The swiotlb code simply allocates a large chunk of low memory early in the bootstrap process; this memory is then handed out in response to DMA allocation requests. In many cases, use of swiotlb memory involves the creation of "bounce buffers," where data is copied between the driver's buffer and the device-accessible swiotlb space. Memory used for the swiotlb is removed from the normal Linux memory management mechanism and is, thus, inaccessible for any use other than DMA buffers. For these reasons, the swiotlb is seen as, at best, inelegant. 

It is also, however, a solution which happens to work. The swiotlb can also accommodate devices with strange DMA masks by searching until it finds memory which fits. So the solution to the problem experienced by nVidia (and others) is likely to be a simple expansion of the swiotlb space. Carving a 128MB array out of main memory for full-time use as DMA buffers may seem like a shocking waste, but, if you have enough memory that you're having trouble with addresses requiring more than 32 bits, the cost of a larger swiotlb will be hard to notice.
