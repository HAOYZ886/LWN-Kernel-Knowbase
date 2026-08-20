---
title: The future of ZONE_DEVICE
url: https://lwn.net/Articles/1016124/
date: "April 4, 2025"
category: "Memory management-ZONE DEVICE"
author: "By Jonathan Corbet April 4, 2025 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 4, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Alistair Popple started his session at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit by proclaiming that `ZONE_DEVICE` is ""the ugly stepchild"" of the kernel's memory-management subsystem. Ugly or not, the ability to manage memory that is attached to a peripheral device rather than a CPU is increasingly important on current hardware. Popple hoped to cover some of the challenges with `ZONE_DEVICE` and find ways to make the stepchild a bit more attractive, if not bring it into the family entirely. 

There are five different types of memory managed under `ZONE_DEVICE`; for the curious, they are: 

  * `MEMORY_DEVICE_PRIVATE`: device-hosted memory that is not directly accessible by the CPU. 
  * `MEMORY_DEVICE_COHERENT`: memory that _is_ directly accessible and maintains cache coherency on both the CPU and device sides. CXL memory is one example of this type. 
  * `MEMORY_DEVICE_FS_DAX`: memory set aside for use with the [DAX](<https://docs.kernel.org/filesystems/dax.html>) (direct file access) subsystem. 
  * `MEMORY_DEVICE_GENERIC`: relatively normal-looking memory, hosted on a device, that is often used for DAX as well. 
  * `MEMORY_DEVICE_PCI_P2PDMA`: memory accessible on the bus used for direct memory transfers between devices. 

About the only thing these types have in common is that this memory is not allocated directly by the memory-management subsystem. Popple is most interested in the device-private type, associated with devices like GPUs. This memory, being controlled by the relevant driver, has a lifetime that is tied to that of the driver. It may be possible to map it into user space on demand, but, since it is not normal memory, it cannot be tracked on the kernel's least-recently-used lists. As a result, drivers have used the `lru` field of the `page` structure, which is destined for eventual removal, for other purposes. 

[![\[Alistair Popple\]](https://static.lwn.net/images/conf/2025/lsfmm/AlistairPopple-sm.png)](<https://lwn.net/Articles/1016132/>) Some of Popple's recent work in this area has been to improve the reference counting of `ZONE_DEVICE` pages; most types are now properly tracked this way. One problem, which took 32 patches to fix, was that FS-DAX pages had reference counts that were off by one, meaning that a reference count of one actually meant that a page was free. That has (as of 6.15) been fixed, freeing a page bit in the process. 

Having solved that problem, Popple is wondering what to be working on next. He would like to support huge device-private pages; only the DAX types of `ZONE_DEVICE` pages support anything larger than base pages now. His cleanup work should enable the use of huge pages more widely. There was some discussion on just how folios might be created for larger `ZONE_DEVICE` pages. Meanwhile, Balbir Singh has posted [a patch series](<https://lwn.net/ml/all/20250306044239.3874247-1-balbirs@nvidia.com/>) adding transparent huge page support for device-private pages. 

Currently, Popple said, drivers are using the [`migrate_vma` interface](<https://elixir.bootlin.com/linux/v6.13.7/source/include/linux/migrate.h#L192>) to move data to and from device memory. It is a simple interface, working with arrays of page-frame numbers, but it gets increasingly unwieldy as page sizes get larger. He would like to enable the splitting and merging of huge pages on migration; huge pages are often easily available on the device side. 

A bigger problem, he said, is enabling file-backed, device-private pages; currently, those pages only work as private, anonymous memory. It would be nice, he said, to be able to just pass a virtual address to a device and let it access whatever contents are there. This ""sort-of works"" with file-backed data on the host, but those pages cannot be migrated to the device. So memory has to be accessed remotely over the bus, which is slower. Things would work a lot better if devices could work with file-backed pages locally. 

Matthew Wilcox asked what the use case was for file-backed device-private pages. The answer was, inevitably, working with AI training data. 

Popple continued that he is looking at letting device-private pages exist in the page cache. There are only a few lookup functions for the page cache, so it should be relatively easy to create special cases for device-private pages. The read side is especially easy, the data can just be reread from storage. For anything requiring writing to shared data, the kernel would handle a page fault, then call into the driver to put the relevant pages back into host memory. 

David Hildenbrand said that this scheme did not sound entirely crazy, and asked how many hoops Popple was having to jump though to implement this; Popple said there weren't that many. As more information moves into folios, he said, the task will get even easier in the future. 

Wilcox was more dubious about the shared case. Shared, writable mappings are ""a terrible programming model"", he said, and the error-handling is difficult. ""What if we just don't do any of this?"". To migrate data to a device, let the device just have the copy, and mark it accordingly on the host. If something has to be done on the host side, just invalidate the device's copy, he said, but do not ever let devices write to this data. If writes absolutely must happen, they can be done over the bus; it will be slow, but it really just has to work. He wondered whether anybody writes to training data on the device side. 

Josef Bacik concurred, saying that he has recently spent a lot of time working with AI models; his opinion on allowing shared writable mappings was: ""this whole thing will suck"". Nobody wants to copy data into user space, then to the device; they would much rather copy the data directly to the device and avoid allocating host memory at all. Managing sharing will be difficult, he said; the implementation should be as simple as possible, and stop at providing read-only access. 

Hildenbrand suggested using the kernel's MMU notifiers to catch faults on shared memory as a way of catching host-side write attempts. It might be worth the trouble he said; but perhaps it is easier to just say that the device will not see host-side modifications. Popple said that in such cases it is better to invalidate pages on the device for correctness, even if the result is slow. 

Bacik closed the session with what seemed like a consensus view. Kernel developers want to anticipate all possible use cases, he said, but sometimes it is better to stop with implementing what can be done correctly. Writing to data shared in device-private pages is not a use case to worry about; any use case that might be proposed would have to be ""pretty compelling"" to be considered. Otherwise, he said, the only reasonable course is to stick with the simpler case that can be implemented correctly.
