---
title: Unaddressable device memory
url: https://lwn.net/Articles/717614/
date: "March 22, 2017"
category: "Memory management-Heterogeneous memory management"
author: "By Jake Edge March 22, 2017 LSFMM 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
March 22, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

In a morning plenary session on the first day of the 2017 Linux Storage, Filesystem, and Memory-Management Summit, Jérôme Glisse led a discussion on memory that cannot be addressed by the CPU because it lives in devices like GPUs or FPGAs. There is often a substantial pile of memory on these devices and it can be accessed much more quickly by the devices than the system RAM can be. Making it easier for user-space programmers to use that memory transparently is the goal of the [heterogeneous memory management (HMM) patches](<https://lwn.net/Articles/679300/>) that Glisse has been working on. 

GPUs and FPGAs can have sizable chunks of memory (8-64GB currently, he said) onboard; currently that memory is accessed by using `mmap()` on the device file. That was fine for graphics (e.g. OpenGL) and for the first version of the OpenCL heterogeneous processing framework, but it does result in a split address space. The devices can access both the system memory and their own local memory, but the CPU cannot really use the device memory because it is not cache-coherent and does not support atomic operations. 

[ ![\[Jérôme Glisse\]](https://static.lwn.net/images/2017/lsfmm-glisse-sm.jpg) ](<https://lwn.net/Articles/717655/>)

The programs written to run on these devices use `malloc()` for their data structures; they need to pin that system memory so that the GPU or FPGA can access them. Programs can, instead, replicate their data structures into the device's memory, but it is cumbersome and bug prone to do so. What is needed is a shared address space where system memory can be migrated to the device memory transparently, he said. 

This shared address space is becoming an industry standard; Windows can do it and C++ standards require it in order to use device memory. OpenCL version 2.0 and beyond need it as does the CUDA parallel programming framework. Handling the memory migration transparently allows programmers to write code that does not have to be aware of whether it is running on the GPU or not. 

Today, GPU memory bandwidth is 1TB per second, while PCIe is 32GB per second and system memory can be accessed at 80GB per second (with four-channel DDR memory). The GPU can access its data much more quickly, so the CPU becomes the bottleneck for memory access. 

There is the possibility of a hardware solution, but no common hardware today can present the device memory as regular memory that is cache-coherent and supports atomic operations from both the device and the CPU. So a software solution is needed, he said. 

HMM is using the `ZONE_DEVICE` allocation zone type but in a different way than was [presented](<https://lwn.net/Articles/717555/>) in the previous plenary. The device memory is tagged as `ZONE_DEVICE`, but system memory is allowed to migrate there. From the CPU perspective it is like swapping the memory to disk; if it needs to access the page, the CPU will get a page fault and has to migrate the memory back from the device. 

Glisse has two possible solutions in mind to provide the shared address space. The first would protect system memory pages so that they cannot be read or written. The pages would then be duplicated on the device and all reads and writes from the CPU would be intercepted. 

Someone asked about the size of the migrations, noting that doing 4KB at a time would not work well. Glisse said that in a typical situation it would be at least a few megabytes and that common use cases would migrate 10GB to the device. The GPU would crunch on that data and then migrate back the results. In between, there would normally be no access to that memory from the CPU. It is definitely dependent on the workload but avoiding migration of 4KB pages one at a time is important. 

There is a potential for pages ping-ponging between CPU memory and device memory, which also needs to be avoided. He is aware of that problem and believes that in the long run drivers will track that kind of access and not migrate pages to the device if the CPU accesses them. Mel Gorman noted that while there is the potential for memory to ping-pong between the two, it is an application bug if it happens. 

Dave Hansen pointed out that `malloc()` will allocate memory for things that should be migrated together with other data that should not be. Glisse acknowledged that, but there will be some pages that simply will not migrate to the device, which is not a big problem since they still can be accessed by the device from the system memory. There also some tools available to help do the right kind of allocations to avoid the problem. 

One problem with mirroring data on the device is that the memory is duplicated. A different idea would be to migrate the memory completely to `ZONE_DEVICE` pages on the device that are not accessible to the CPU at all. Any read or write to the memory from the CPU will trigger a migration back. That requires catching any kind of read or write, including from system calls or direct I/O. 

Al Viro pointed out that the system call or direct I/O level is not the right level to intercept the operations that require a migration back to the CPU. Rather the interception should be done at calls to `get_user()` and `put_user()`. If you catch all accesses from user space, that will ensure that all of the reads and writes to the memory are handled, he said. 

Dan Williams was concerned that Glisse is using `ZONE_DEVICE` differently than it is being used for persistent-memory arrays. He wondered how the two uses could be differentiated. His patches are able to distinguish between the two uses, Glisse said. 

Glisse then moved into how to protect a page from reads and writes. He suggested that it could be done by making what the [kernel shared memory (KSM)](<https://lwn.net/Articles/330589/>) feature is doing "more generic". For pages that need to be protected, the `page->mapping` entry would be replaced with a pointer to a protection structure and all dereferences of `page->mapping` would be wrapped with a helper function. 

He has patches to make that change that were generated using Coccinelle, so they are simply mechanical changes. It is good to reuse the existing KSM mechanism, he said, but wondered if there would be impacts elsewhere from the changes. Someone from the audience said that the wrapper would be useful for other reasons as well, however. Glisse said he would push those patches soon to make it available for those other uses. 

There is a need to do writeback from the device memory pages and Glisse suggested using the ISA block queue bounce buffer code "from the 90s" that is still in the kernel. But James Bottomley pointed out that bounce buffers are used for more than just ISA devices; unaligned `struct bio` writes also use them. The upshot is that code that is being used now will definitely work for what Glisse is trying to do. Glisse would also like to give the system administrator some sort of knob to control writeback from device memory, so that those who want to squeeze out every last bit of performance (at the risk of filesystem integrity) can do so. 

An audience member suggested that kernel developers were coming to a "shared agreement" about how the kernel should refer to memory that is not directly addressable. That kind of memory would get page structures, but it would not be directly accessible from the kernel. There seemed to be general assent to that idea. 

There are a number of corner cases that Williams is concerned about and he is still not convinced that reusing `ZONE_DEVICE` for HMM makes sense. Glisse said he could add a new zone type if needed. Gorman is not particularly concerned about the corner cases, for now; Glisse agreed, saying that if those cases become problems, they can be looked at then. 

There are no upstream users of the HMM feature, but Glisse expects that the Nouveau driver will use it, as will AMD GPU drivers and various FPGA drivers down the road. Users will need to be aware that the system memory will need to be larger than all of the device memory available (or used); otherwise the system could livelock when migrating results back from the device memory, Gorman said. Glisse agreed that it could be a problem, but the patches will be useful to see how big of a problem it is in practice. 

Gorman thinks the focus should be on getting HMM into the kernel; interactions with filesystems (i.e. writeback) can be handled down the road. Glisse agreed, saying that he is working on getting HMM into the kernel as it is today, but wanted to discuss some of the other issues to try to make sure that the ideas he has for dealing with filesystem interaction are not way off base.
