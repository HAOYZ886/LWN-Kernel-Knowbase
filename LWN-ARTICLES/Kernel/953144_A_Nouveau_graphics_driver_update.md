---
title: A Nouveau graphics driver update
url: https://lwn.net/Articles/953144/
date: "December 1, 2023"
category: "Device drivers-Graphics"
author: "By Jonathan Corbet December 1, 2023 LPC"
---

By **Jonathan Corbet**  
December 1, 2023

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2023-Linux_Plumbers_Conference>)

Support for NVIDIA graphics processors has traditionally been a sore point for Linux users; NVIDIA has not felt the need to cooperate with the kernel community or make free drivers available, and the reverse-engineered Nouveau driver has often struggled to keep up with product releases. There have, however, been signs of improvement in recent years. At the [2023 Linux Plumbers Conference](<https://lpc.events/event/17/page/198-lpc-2023-overview>), graphics subsystem maintainer Dave Airlie provided an update on the state of support for NVIDIA GPUs and what remains to be done. 

The kernel community's relationship with NVIDIA "has gone up and down" over the years, Airlie began. Recently, though, the company has rearchitected its products, adding a large RISC-V processor (the GPU system processor, or GSP) and moving much of the functionality once handled by drivers into the GSP firmware. The company allows that firmware to be used by Linux and shipped by distributors. This arrangement brings a number of advantages; for example, it is now possible for the kernel to do reclocking of NVIDIA GPUs, running them at full speed just like the proprietary drivers can. It is, he said, a big improvement over the Nouveau-only firmware that was provided previously. 

[![\[Dave Airlie\]](https://static.lwn.net/images/conf/2023/lpc/DaveAirlie-sm.png)](<https://lwn.net/Articles/953147/>) There are a number of disadvantages too, though. The firmware provides no stable ABI, and a lot of the calls it provides are not documented. The firmware files themselves are large, in the range of 20-30MB, and two of them are required for any given device. That significantly bloats a system's `/boot` directory and initramfs image (which must provide every version of the firmware that the kernel might need), and forces the Nouveau developers to be strict and careful about picking up firmware updates. 

Nouveau work has taken a bit of a setback since longtime developer Ben Skeggs left the project, but he did manage to do a lot of refactoring before he went. Nouveau now has initial GSP support for one firmware version; that code was merged in for the 6.7-rc1 release. It is only enabled for the [Ada](<https://www.nvidia.com/en-us/geforce/ada-lovelace-architecture/>) series of GPUs by default; with a command-line argument it can be made to work with [Turing](<https://www.nvidia.com/en-us/geforce/turing/>) and [Ampere](<https://www.nvidia.com/en-us/data-center/ampere-architecture/>) devices as well. It is missing some features, including fault handling (which "shouldn't be too hard" to add) and sensor monitoring, which doesn't work at all. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

NVIDIA's firmware, Airlie said, comes with a set of include files that, in turn, define structures that change over time. To deal with these changes, the driver is going to need some sort of automated ABI generation; he noted that the developers working on the Apple M1 GPU driver have run into the same problem. This problem could be made easier to tackle, he suggested, if the driver were, like the M1 driver, to be rewritten in Rust. 

#### Next steps

Supporting the GSP firmware is just the beginning, though; at this point, Airlie took a step back and talked about the task of making a useful GPU driver in general. Years ago, a graphics card came with some video RAM and a [graphics translation table (GTT)](<https://en.wikipedia.org/wiki/Graphics_address_remapping_table>). The driver would map system memory into the graphics card; user space could then submit buffer handles that would be relocated for the graphics device. This approach works, he said, but it is slow. 

Current GPUs have full virtual memory, instead, which saves a lot of that overhead. The kernel has grown a number of subsystems for working with this virtual memory, including the [graphics execution manager (GEM)](<https://docs.kernel.org/gpu/drm-mm.html#the-graphics-execution-manager-gem>) for buffer-object management, the [translation table manager (TTM)](<https://docs.kernel.org/gpu/drm-mm.html#the-translation-table-manager-ttm>) for discrete video-RAM buffer-object management, and a bunch of synchronization and fencing code. Initially, the DRM subsystem would tie the allocation of a buffer to an allocation of virtual memory at the same time; that was easy to do and sufficed to implement OpenGL. But, he said, the graphics world moved on from there. 

Specifically, [Vulkan](<https://www.vulkan.org/>) came along. It brought the concept of sparse memory and, with it, virtual memory that is managed by user space. Vulkan can handle both synchronous and asynchronous virtual-area updates, but it "gets complicated". Various drivers started inventing their own virtual-area management; as a way of bringing that work back together, the [VM_BIND](<https://www.kernel.org/doc/html/latest/gpu/drm-vm-bind-async.html>) API was developed. 

This is consistent with a recurring pattern, Airlie said. The DRM developers work to share common code between graphics drivers, but the driver developers keep trying to reinvent wheels, a tendency that has to be resisted. The subsystem did well with regard to mode setting, he said, but less well on the acceleration side; there is a "common GPU scheduler" that is only used by one driver, for example. Similarly, there are a lot of drivers implementing VM_BIND by doing their own virtual-area management. 

In response, Airlie came up with the "good idea" of getting somebody else to write a common virtual-area manager, called GPUVM, inspired by the amdgpu code. It is intended to be useful for all drivers; it is used by the Nouveau, Xe (Intel's new driver), and Panfrost drivers now. Hopefully the amdgpu and MSM drivers will pick it up as well. The best part is that there are multiple developers who understand it and can help to keep it from going off in the wrong direction. GPUVM has been through a lot of iterations, he said, providing "lots of learning experiences". 

As an example, he talked about the problem of fence signaling. A fence indicates when a series of GPU operations has been completed; waits for these fences have to be time-bounded, or the memory-management subsystem might deadlock. In short, a GPU can easily pin down all of a system's RAM if given the opportunity. There is a shrinker that can be called when memory gets tight, but it will have to wait for fences to be signaled to know when memory can be freed. If the code that set the fence decides to allocate more memory while this is happening, a deadlock results. To avoid this outcome, developers have to strictly limit the operations that can be performed in fence-signaling critical sections; care must also be taken before acquiring any locks. It would be nice to be able to update the page tables during this code, but that ran into deadlock problems and had to be backed out. 

Returning to Nouveau, Airlie said that the initial VM_BIND API, using GPUVM, synchronous objects, and integration with the scheduler, was merged for the 6.6 release. There are a lot of improvements in the works that should land in 6.8. At this point, he said, we have the core of a modern GPU driver for NVIDIA hardware — for graphics, at least. More work will be required before Nouveau can support compute applications. 

On the user-space side, Faith Ekstrand has been developing the NVK Vulkan driver for Nouveau; this driver recently [reached Vulkan 1.0 conformance](<https://www.collabora.com/news-and-blog/news-and-events/nvk-reaches-vulkan-conformance.html>). This work involved creating a new compiler, called NAK, that has just been merged into Mesa; this compiler yields far better performance (from 20 frames per second to over 1000) than the old "codegen" compiler did. Naturally, this compiler is written in Rust. The next step, Airlie concluded, is to move forward to Vulkan 1.3. 

[Video](<https://www.youtube.com/live/LipsVK5d_vM?si=MvJmtsFKbpa5JyqC&t=326m21s>) and [slides](<https://lpc.events/event/17/contributions/1505/attachments/1315/2641/Nouveau%20GSP%20GPU%20VA%20management.pdf>) from the talk are available. 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting our travel to this event.]
