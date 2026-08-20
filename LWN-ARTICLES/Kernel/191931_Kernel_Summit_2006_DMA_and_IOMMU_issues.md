---
title: "Kernel Summit 2006: DMA and IOMMU issues"
url: https://lwn.net/Articles/191931/
date: "July 19, 2006"
category: Direct memory access; IOMMU
author: 
---

[2006 Kernel Summit coverage](<https://lwn.net/Articles/KernelSummit2006/>) on LWN.net.   
---  
Kumar Gala talked about the use of DMA engines, which are becoming regular features on current processors. These engines can perform simple memory operations, such as zeroing and copying, offloading that work from the host CPU. The more advanced engines can perform transformations on data, all the way up to those which can handle cryptographic operations. Nobody argues that these engines should not be supported; the main issue is what sort of API should be created to access them. 

The initial discussion involved API calls for allocating DMA engine channels and submitting operations to them. After some discussion, however, it was agreed that this was the wrong approach. Nobody wants to see the kernel fill up with code which checks for DMA engines, attempts to allocate channels, and codes around failures. Far better would be to have a function which arranges for a copy operation to happen using the best method available at the moment. An asynchronous interface, with a callback to indicate completion, is probably the best way to go, though there are some issues to work out there. 

James Bottomley talked about a related issue: the management of I/O memory management units (IOMMUs). An IOMMU provides a virtual address space to DMA-capable devices, solving addressing issues and setting up transparent scatter/gather operations. Not all architectures have IOMMUs, but that may be about to change. 

The driving force at this point is virtualization; evidently there is a great deal of interest in assigning devices to virtualized systems and letting those systems handle the I/O details. If you give a DMA-capable device to a virtualized host, however, you give that host an engine which is capable of overwriting any device-addressable memory on the system. That is a violation of the isolation model and a potential security problem One could solve this problem by not letting virtualized hosts program DMA operations, but the preferred approach seems to be to restrict those operations by way of an IOMMU. 

Making that sort of restriction work will require some changes to the kernel's DMA interface. The current DMA mapping interface, which is designed to be lightweight and fast, will have to become a trap into the hypervisor, which can then police the IOMMU settings. As a result, multi-chunk DMA operations will, whenever possible, need to be mapped in a single operation to avoid causing excessive traps. That means using `dma_map_sg()`, rather than mapping each page individually. The block layer, says James, works that way now, but the networking code does not. That will need to be fixed, perhaps by way of unifying some of the scatter/gather I/O paths in the kernel. 

Life gets even harder when trying to share devices between virtual machines \- a use case for which there is, apparently, some real demand. Nobody really knows how to do that, not even the hardware vendors. If the Linux developers would like to have any influence over how this mode of operation is to be controlled, now is the time to come up with proposals. James will (reluctantly) work to bring such a proposal about. 

  * [Next: Development Process II](<https://lwn.net/Articles/191932/>)
