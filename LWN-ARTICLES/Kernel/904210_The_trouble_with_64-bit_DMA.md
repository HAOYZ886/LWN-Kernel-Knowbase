---
title: "The trouble with 64-bit DMA"
url: https://lwn.net/Articles/904210/
date: "August 11, 2022"
category: "Direct memory access; Releases-6.0"
author: "By Jonathan Corbet August 11, 2022"
---

By **Jonathan Corbet**  
August 11, 2022

We live in a 64-bit world, to the point that many distributors want to stop supporting 32-bit systems at all. However, lurking within our 64-bit kernels is a subsystem that has not really managed to move past 32-bit addresses. The quick merge-window failure of an attempt to use 64-bit addresses in the I/O memory-management unit (IOMMU) subsystem shows how hard it can be to leave all of one's 32-bit history behind. 

Peripheral devices that move data at any significant rate have to support direct memory access (DMA) to get reasonable performance. As the DMA name suggests, these devices once had direct access to the system's memory in the physical address space. Over time, though, most systems have moved to interposing an IOMMU between devices and memory, for a number of reasons. The IOMMU can help to ensure that the device only accesses the memory that was intended for it, for example. It is also possible to use the IOMMU to make pages scattered throughout physical memory appear to be contiguous from the device's point of view. 

For all of this to work, a device driver must create an IOMMU mapping for an I/O buffer before presenting the mapped addresses to the device. Those addresses, called I/O virtual addresses (or IOVAs), look like physical addresses, but they have their own 64-bit address space. One would expect to be able to pass an address anywhere in that range to a device, but life is not so simple; many devices have surprising limitations on how many address bits they can actually use. The kernel's DMA-mapping layer takes this into account; drivers pass in a mask indicating the address range that the device can handle, and the kernel finds an address within that range. 

The IOMMU layer imposes an additional constraint, though, in that it will pick an address below 4GB (i.e. one that fits in 32 bits) if at all possible. In the early days of the PCI bus, a device performing DMA to a 64-bit address had to use a special "dual-address cycle" (DAC) mode with each access; DAC cycles were slower than single-address cycles, and a lot of devices either didn't implement them at all or had buggy implementations. Limiting IOVAs to the 32-bit range helped performance and danced around the ever-present possibility of hardware bugs. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

It is now 2022, and the PCI bus has been superseded by PCI-Express, which does not have the same performance problems with DAC addresses. One might think that current hardware would not have trouble with 64-bit addresses, which are not exactly new technology at this point. The 32-bit constraint is still in place, though, and it is causing some pain of its own. Back in June, Robin Murphy posted [a patch](<https://lwn.net/ml/linux-kernel/3f06994f9f370f9d35b2630ab75171ecd2065621.1654782107.git.robin.murphy@arm.com/>) describing that pain: 

> The IOVA allocator doesn't behave all that well once the 32-bit space starts getting full. As DMA working sets get bigger, this optimisation increasingly backfires and adds considerable overhead to the dma_map path for use-cases like high-bandwidth networking. We've increasingly bandaged the allocator in attempts to mitigate this, but it remains fundamentally at odds with other valid requirements to try as hard as possible to satisfy a request within the given limit. 

At a first glance, 4GB of DMA address space seems like it should be enough for anybody, but a big system with the right workload can fragment that space and make allocations hard. On the theory that the kernel is needlessly restricting its options to satisfy constraints that no longer make sense, Murphy changed the default so that the IOMMU layer would no longer try to find a 32-bit-compatible address and would, instead, use the full address range that the target device claimed to support. That makes the performance problem go away, which is a good thing. 

The problem of buggy devices, though, cannot be made to disappear with a simple kernel patch. In a sense, that problem is even worse now, in that the 32-bit constraint may have papered over bugs in both devices and the drivers that control them for years. A driver author, perhaps an inexperienced developer who has not yet learned about the mendacity of hardware data sheets, may have trusted the documentation and told the DMA-mapping layer that their hardware could handle full 64-bit IOVAs when, in fact, it cannot. Now the only thing making that hardware actually work is the 32-bit constraint applied by the IOMMU layer. 

Murphy acknowledged the risk that this change would expose this kind of bug; the patch included a couple of options for restoring the old behavior. But Murphy wanted to push the change through: 

> Let's be brave and default it to off in the hope that CI systems and developers will find and fix those bugs, but expect that desktop-focused distro configs are likely to want to turn it back on for maximum compatibility. 

IOMMU maintainer Joerg Roedel [applied the patch](<https://lwn.net/ml/linux-kernel/YrMSJ6AGwn3PxSIH@8bytes.org/>) with reservations: ""I don't have an overall good feeling about this, but as you said, let's be brave"". The patch then [landed in the mainline](<https://git.kernel.org/linus/4bf7fda4dce2>) during the 6.0 merge window. 

It didn't stay there for long, though. One of the core rules of kernel development is that good things rarely result from breaking Linus Torvalds's machine, and that is what happened here. He promptly [reverted the change](<https://git.kernel.org/linus/af3e9579ecfb>), saying: ""It turns out that it was hopelessly naive to think that this would work, considering that we've always done this. The first machine I actually tested this on broke at bootup"". He [added](<https://lwn.net/ml/linux-kernel/CAHk-%3DwiSxEVW78nTumvz%2BZYt2MZcmqe6S0nb_-efKYm636VuGA%40mail.gmail.com/>) that Murphy could ""try again in a decade or so"". 

The problem, of course, is that the problems created by the 32-bit constraint are unlikely to get better by themselves in the next decade or so. There is going to be increasing pressure to leave that behavior behind, at least on machines where the hardware is known to work properly. Somehow, the community is going to have to find a way to change things that doesn't break systems across the planet. Perhaps drivers could set a new flag for hardware that is known to be good, or perhaps some sort of list could be maintained separately. The kernel has spent years papering over buggy hardware and drivers; climbing out of the resulting hole is likely to take a while as well.
