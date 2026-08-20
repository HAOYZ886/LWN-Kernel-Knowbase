---
title: Restricted DMA
url: https://lwn.net/Articles/841916/
date: "January 7, 2021"
category: Direct memory access
author: "By Jonathan Corbet January 7, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 7, 2021

A key component of system hardening is restricting access to memory; this extends to preventing the kernel itself from accessing or modifying much of the memory in the system most of the time. Memory that cannot be accessed cannot be read or changed by an attacker. On many systems, though, these restrictions do not apply to peripheral devices, which can happily use direct memory access (DMA) on most or all of the available memory. The recently posted [restricted DMA patch set](<https://lwn.net/ml/linux-kernel/20210106034124.30560-1-tientzu@chromium.org/>) aims to reduce exposure to buggy or malicious device activity by tightening up control over the memory that DMA operations are allowed to access. 

DMA allows devices to directly read from or write to memory in the system; it is needed to get reasonable I/O performance from anything but the slowest devices. Normally, the kernel is in charge of DMA operations; device drivers allocate buffers and instruct devices to perform I/O on those buffers, and everything works as expected. If the driver or the hardware contains bugs, though, the potential exists for DMA transfers to overwrite unrelated memory, leading to corrupted systems and unhappy users. Malicious (or compromised) hardware can use DMA to compromise the system the hardware is attached to, making users unhappier still; [examples of this type of attack](<https://blade.tencent.com/en/advisories/qualpwn/>) have been posted over the years. 

One way to address this problem is to place an I/O memory-management unit (IOMMU) between devices and memory. The kernel programs the IOMMU to allow access to a specific region of memory; the IOMMU then keeps devices from straying outside of that region. Not all systems are equipped with an IOMMU, though; they are mostly limited to the larger processors found in desktop machines, data centers, and the like. Mobile systems usually lack an IOMMU. 

The restricted DMA patch set, posted by Claire Chang, is an attempt to apply some control to DMA operations on systems without an IOMMU. To do so, it builds on an old, relatively obscure kernel mechanism called the "swiotlb", which stands for "software I/O translation lookaside buffer". The swiotlb was originally created to facilitate operations with devices that have annoying DMA limitations, such as the inability to address all of the memory in the system. The core mechanism used within the swiotlb is bounce buffering: allocating a buffer in a region that the device in question is able to access, then copying data between I/O buffers and this bounce buffer as needed. Copying the data clearly slows I/O operations, but it is far better than not using DMA at all. 

Chang's patch set enhances the swiotlb by allowing it to allocate a specific range of physical memory and associate it with a given device; this range can be specified in a devicetree using the new `restricted-dma-pool` ["compatible" property](<https://elinux.org/Device_Tree_Usage#Understanding_the_compatible_Property>). All DMA operations involving that device will be bounced through that range of memory, effectively isolating devices from the actual I/O buffers seen by the rest of the system. 

Using this kind of bounce-buffering offers some benefit on its own. Your editor, who has written device drivers in the past, would never have committed such an error, but it is not unheard of for driver bugs to result in a device performing DMA when the rest of the system thinks it should be idle. Having memory buffers seemingly randomly overwritten in unreproducible ways can (again, your editor relies on the word of others for this) result in corrupt data, painful debugging sessions, and excessive alcohol use. By separating the buffer used by the device from the buffer used by the kernel, restricted DMA can mitigate many of the more unpleasant effects of this sort of bug. 

Readers may be wondering, though, how the use of the swiotlb will protect the system against a malicious or compromised device; such devices may well ignore polite requests to restrict their DMA activities to the designated area, after all. The answer is that it will not protect systems from this type of attack — at least, not on its own. The evident intent, though, is to pair restricted DMA with [trusted firmware](<https://www.trustedfirmware.org/about/>) implementations that are able to restrict DMA operations to specific ranges of memory; these restrictions are set up at (or before) boot time and cannot be changed by the kernel. So the trusted firmware can constrain a device's access to the designated region, while the restricted DMA mechanism causes all DMA operations to go through that region. Together, these mechanisms provide a way to enable DMA without allowing a device to access arbitrary memory, all without an IOMMU in the system. 

The amount of setup work required suggests that this capability will not be present on most general-purpose systems anytime soon. But on tightly controlled systems — mobile devices, for example — there is clear value in making the additional effort to prevent compromise via a hostile device. It's not clear whether the restricted DMA patches will make it into the mainline in their current form, but chances are that this kind of mechanism will be merged sooner or later.
