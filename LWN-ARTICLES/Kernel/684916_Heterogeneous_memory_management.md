---
title: Heterogeneous memory management
url: https://lwn.net/Articles/684916/
date: "April 27, 2016"
category: "Memory management-Heterogeneous memory management"
author: "By Jonathan Corbet April 27, 2016 LSFMM 2016"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

The processor that one thinks of as "the" CPU is not the only processor on most systems; indeed, it is often not the fastest. Attached devices, first and foremost the graphics processor (GPU), have their own processors that can speed a number of computing tasks. They often have full access to system memory, but there are obvious challenges to sharing that memory completely between the CPU and other processors. The heterogeneous memory management (HMM) subsystem aims to make that sharing possible; Jérôme Glisse led a session on HMM for the memory-management track at the 2016 Linux Storage, Filesystem, and Memory-Management Summit. 

The key feature of HMM, Jérôme said, is making it possible to mirror a process's address space within the attached processor. This should happen without the need to use a special allocator in user space. On the hardware side, there are a couple of technologies out there that make this mirroring [![\[Jérôme Glisse\]](https://static.lwn.net/images/conf/2016/lsfmm/JeromeGlisse-sm.jpg)](<https://lwn.net/Articles/684920/>) easier. One is the PowerPC CAPI interface; another is the PASID mechanism for the PCI Express bus. On the software side, options are to either mirror the CPU's page table in the attached processor, or to migrate pages back and forth between CPU and device memory. Regardless of how this is done, the hope is to present the same API to user space. 

We care about this, Jérôme said, because the hardware is out there now; he mentioned products from Mellanox and NVIDIA in particular. Drivers exist for this hardware which, he said, is expensive at the moment, but which will get cheaper later this year. If we don't provide a solution in the kernel, things will run much more slowly and will require the pinning of lots of memory. It will be necessary to add more memory-management unit (MMU) notifiers to device-driver code, which few see as desirable. [OpenCL](<https://www.khronos.org/opencl/>) support will only be possible on integrated GPUs. In general, he said, it is better to support this capability in the kernel if possible. 

The solution to these ills is the HMM patch set, which provides a simple driver API for memory-management tasks. It is able to mirror CPU page tables on the attached device, and to keep those page tables synchronized as things change on the CPU side. Pages can be migrated between the CPU and the device; a page that has been migrated away from the CPU is represented by a special type of swap entry — it looks like it has been paged out, in other words. HMM also handles DMA mappings for the attached device. 

Andrew Morton noted that the patch set is "a ton of code," which always makes it harder to get merged. There was some talk of splitting the patch set into more palatable pieces; some of the code, evidently, is also useful for KVM virtualization. Andrew told Jérôme to take care to document who the potential users of this code are. Then, he said, "it's a matter of getting off our asses and reviewing the code." There might be trouble, he said, with the use of MMU notifiers, since Linus has been clear about his dislike of notifiers in the past. 

Overall, though, no objections to the core model were expressed. The HMM code has been in development for some years; maybe it is finally getting closer to inclusion into the mainline kernel.
