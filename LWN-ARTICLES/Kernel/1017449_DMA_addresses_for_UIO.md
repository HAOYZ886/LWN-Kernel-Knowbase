---
title: DMA addresses for UIO
url: https://lwn.net/Articles/1017449/
date: "April 22, 2025"
category: "Device drivers-In user space"
author: "By Jonathan Corbet April 22, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
April 22, 2025

The [Userspace I/O (UIO) subsystem](<https://docs.kernel.org/driver-api/uio-howto.html>) was first [added to the kernel](<https://git.kernel.org/linus/beafc54c4e2f>) by Hans J. Koch for the 2.6.23 release in 2007. Its purpose is to facilitate the writing of drivers (mostly) in user space; to that end, it provides access to a number of resources that user-space code normally cannot touch. One piece that is missing, though, is DMA addresses. [A proposal to fill that gap](<https://lwn.net/ml/all/20250410-uio-dma-v1-0-6468ace2c786@bootlin.com>) from Bastien Curutchet is running into some opposition, though. 

While UIO drivers reside mostly in user space, they still require a small in-kernel component; it is essentially a bit of glue that can probe for a device and handle interrupts, and which informs the kernel of any memory-mapped I/O (MMIO) regions that should be made available to user space. Showing its age, UIO also has a mechanism to make x86 I/O ports available. Once that module is loaded, a user-space driver can open the appropriate `/dev/uioX` device, map the memory regions into its address space, use that mapping to program the device, and read from the device file descriptor to wait for interrupts. The mapped MMIO areas are usually all that is needed to operate the device. 

See [this 2007 article](<https://lwn.net/Articles/232575/>) for an overview of what the UIO interface looked like then and some of the discussions around its addition to the kernel. 

UIO drivers are generally relatively simple; as the complexity of a device increases, so do the advantages of driving it entirely within the kernel. Among other things, UIO drivers are normally limited to simple programmed I/O, where all data transferred goes through the CPU. The performance of programmed I/O transfers is limited, though, so even relatively low-end devices now support DMA transfers instead. Access to the MMIO region is usually sufficient to program a DMA operation into a device, but there is one little detail missing: the address of the I/O buffer to give to the device. 

Coming up with DMA addresses is not quite as easy as it sounds. A driver running in user space will see its virtual address space; it can create a buffer there, but the driver's virtual address for that buffer will mean nothing to the device. Even on the simplest of systems, that address will need to be converted to a physical address that the device can access; the user-space pages involved will also need to be locked into memory for the duration of the transfer. On more complex systems, the "physical address" will be in a separate address space for the bus on which the device resides, and may well be translated through an I/O memory-management unit (IOMMU), which must be appropriately programmed. None of this work can be done from user space. 

According to Curutchet, there are numerous UIO drivers that include hacks to enable the creation and export of DMA addresses for user-space drivers to use. The posted patch series is an attempt to provide a standard, well-tested mechanism to carry out this task in a relatively safe way. The kernel _does_ have some infrastructure for the management of DMA buffers from user space: the [dma-buf layer](<https://docs.kernel.org/driver-api/dma-buf.html>), which was initially created to enable the sharing of DMA buffers between devices. This layer does almost everything that a UIO driver needs, but it does not make the DMA addresses of its buffers available to user space. 

Curutchet's patch set adds a new [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) operation to the dma-buf layer that can provide the DMA address for a dma-buf to the caller. The series also adds a new [dma-buf heaps](<https://docs.kernel.org/next/userspace-api/dma-buf-heaps.html>) module for the creation of simple, cache-coherent DMA buffers. Using this module, user-space drivers can create DMA buffers, but they do not have access to most of the more-complex dma-buf machinery, including all of its synchronization support. 

Christian König repeatedly [requested](<https://lwn.net/ml/all/29eb3698-7dc2-4c32-b636-8ef0ee6d1f47@amd.com>) an open-source driver that uses this new feature. As Thomas Petazzoni [pointed out](<https://lwn.net/ml/all/20250414135228.600c13e6@windsurf>), though, an open-source user has never been required for any other UIO feature. The presence of an open-source consumer for a new interface can be helpful during the review process, but it does seem a bit late to add that requirement here, when it has never been requested before. 

Christoph Hellwig, instead, [said](<https://lwn.net/ml/all/Z_yjNgY3dVnA5OVz@infradead.org>) that the proper solution to this problem was to use the [VFIO](<https://docs.kernel.org/next/driver-api/vfio.html>) and [IOMMUFD](<https://docs.kernel.org/next/userspace-api/iommufd.html>) subsystems. He later [added](<https://lwn.net/ml/all/Z_zwZYBO5Txz6lDF@infradead.org>) that the proposal is ""fundamentally broken and unsafe"" and that, if his suggested approach does not work, the only alternative is to write a full in-kernel driver. Petazzoni [answered](<https://lwn.net/ml/all/20250414102455.03331c0f@windsurf>) that VFIO and IOMMUFD are not suitable for this use case, and [pointed out](<https://lwn.net/ml/all/20250414134831.20b04c77@windsurf>) that making DMA addresses available does not add safety concerns that are not already present: 

> UIO allows to remap MMIO registers into a user-space application, which can then do whatever it wants with the IP block behind those MMIO registers. If this IP block supports DMA, it already means that _today_ with the current UIO subsystem as it is, the user-space application can program a DMA transfer to read/write to any location in memory. 

Creating a clean interface for the management of DMA buffers, he suggested, is likely to make UIO drivers more safe than they are now. Andrew Davis [was not convinced](<https://lwn.net/ml/all/8f55367e-45c0-4280-b1ed-7ce9160c1fad@ti.com>), though, characterizing Petazzoni's argument as: ""UIO is a broken legacy mess, so let's add more broken things to it as broken + broken => still broken, so no harm done"". He later [said](<https://lwn.net/ml/all/4c77a566-d231-43f2-ada6-a81ec6b58237@ti.com/>) that improving the UIO interface ""gives the impression that is how drivers should still be written"" when, he said, the correct solution is to write an in-kernel driver. 

As that last quote shows, the real point of contention here is not whether the provision of DMA addresses will somehow make UIO more dangerous; UIO drivers can already corrupt the system. The question is whether UIO should exist at all in 2025. Support for many classes of drivers in the kernel has improved significantly since UIO was merged, and some developers are undoubtedly unhappy with its potential use in developing proprietary drivers. At this point, to some, the value provided by UIO may appear to be negative. 

UIO cannot be removed, of course, without breaking an unknown number of existing drivers, but it can be frozen in place with the idea of reducing the number of drivers written to use the UIO interface in the future. The real result of denying new UIO features, though, seems likely to be to just push those features into a set of (probably more dangerous) driver-specific hacks, as is evidently already happening for DMA addresses.
