---
title: Presenting heterogeneous memory to user space
url: https://lwn.net/Articles/787326/
date: "May 8, 2019"
category: "Memory management-Heterogeneous memory management"
author: "By Jonathan Corbet May 8, 2019 LSFMM"
---

By **Jonathan Corbet**  
May 8, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Computer memory architecture is growing more complex over time, with different types of memory attached to a CPU via a number of paths. The kernel development community is duly working to make this memory available to user space in an equally diverse set of ways. Two sessions at the 2019 Linux Storage, Filesystem, and Memory-Management Summit presented possible mechanisms and APIs to allow programs to work with the types of memory they need. 

#### The heterogeneous memory mapper

One upcoming development is the "[heterogeneous memory mapper](<https://github.com/westerndigitalcorporation/hmmap>)" (hmmap), which was presented by Adam Manzanares in a combined storage and memory-management session. Hmmap is implemented as a character device that can map different kinds of memory into a process's address space. It is intended to provide both direct and cached access, possibly at the same time (though via different address ranges). Cache management is flexible, with pluggable strategies; the page cache is good for most workloads, Manzanares said, but some may prefer alternatives. The actual cache management to use can be specified by the user. Internally, hmmap is implemented in two layers, one handling caching strategy and one for low-level access to the underlying media. 

Why might one want this? It provides flexibility for different device types, allowing users to mix and match technologies like DAX, RDMA, and basic block devices. It provides the pluggable caching strategies, though he [![\[Adam Manzanares\]](https://static.lwn.net/images/conf/2019/lsfmm/AdamManzanares-sm.jpg)](<https://lwn.net/Articles/787351/>) allowed that he's not fully convinced that this feature is needed. It is an alternative to supporting DAX via NUMA nodes that does not require hardware support. It also provides a clear path to persistent storage, something that is more implicit with the NUMA approach. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Manzanares was quickly accused of reimplementing the page cache, a criticism that he partially accepted. Perhaps, he suggested, some of the features provided by hmmap could be put into the page cache itself instead. James Bottomley noted that one advantage of hmmap is that it can be shrunk more quickly than the normal page cache can be; perhaps a policy could be developed to exploit that capability. 

Mel Gorman said that he was having a hard time seeing the use case for hmmap. Existing kernel functionality, like dm-cache or bcache, can be used already for many storage-acceleration applications. One potentially interesting use case could be a device with directly accessible memory that could be put to use as additional RAM on the system; data could be moved to and from ordinary RAM on demand. This device would differ from others in that the data would not be stored in any persistent media. 

With regard to dm-cache in particular, he said, it seems that it can handle part of this use case; it works like a page cache, moving pages between faster and slower devices. It is inefficient for some kinds of workloads, though, where it ends up touching large amounts of memory. Persistent memory is better for moving smaller amounts of memory efficiently; it could benefit from a software layer that can take advantage of that. 

Other potential uses are harder to lay out. Gorman suggested that Manzanares should enumerate the use cases for this feature and explain why currently available solutions are not good enough. He suggested that hmmap is an implementation of dm-cache with a different API. As the session ended, Manzanares said that he would look more deeply into dm-cache and outline any changes that would be needed to make it more widely applicable. 

#### hbind()

The other proposal was described by Jérôme Glisse in a memory-management track session on the final day. Glisse has been working with heterogeneous memory for some time; his focus at the moment is on memory that is not necessarily directly accessible by the CPU and which may not be cache coherent. Device drivers tend to want complete control over such memory; pinning of pages by the CPU cannot generally be supported. 

There has been a lot of talk about supporting memory types using the NUMA abstraction, Glisse said, but that approach has some limitations. Applications might start using device memory without understanding its limitations, leading to data corruption or other unfortunate consequences. His solution is to make the use of this memory something that applications opt into explicitly, specifying the type of memory they are looking for. 

Applications would do this with the new `hbind()` system call, which was [posted for review](<https://lwn.net/ml/linux-kernel/20181203233509.20671-1-jglisse@redhat.com/>) several months ago. This call exists to unify access to different types of device memory; that access is generally available now, but it requires a different `ioctl()` call for each device type. Rather than forcing applications to support a range of calls, Glisse would [![\[Jérôme Glisse\]](https://static.lwn.net/images/conf/2019/lsfmm/JérômeGlisse2-sm.jpg)](<https://lwn.net/Articles/787854/>) like to provide a single call that works for everything. `hbind()` would be like [`mbind()`](<http://man7.org/linux/man-pages/man2/mbind.2.html>), but it would work with device memory as well as ordinary, CPU-attached memory. 

There were a lot of questions about just what the semantics of this new system call would be; Glisse described it as a request to migrate content to device memory while keeping it accessible. Michal Hocko said that he would rather have a better description of the semantics than the implementation details Glisse was giving. He asked what would happen if multiple users request more memory than a device can provide; how can one guarantee access for the most important process? Glisse replied that the answer would be device-specific. Hocko complained that the interface is insufficiently defined in general; if he has to maintain that API forever, he said, he wants specifics. 

Dave Hansen said that it looked like Glisse is creating a parallel interface to the system's NUMA functionality; is NUMA really not good enough to solve this problem? Aneesh Kumar said, though, that it wasn't possible to allow the kernel to manage this memory, since things that the kernel wants to do (page pinning, for example) cannot be supported. In the end, though, Hansen replied, users want to allocate this memory with `malloc()`, which ends up involving the kernel anyway. 

Glisse talked briefly about how applications would discover which device memory is available on a system. There would be a new directory called `/sys/devices/system/hmemory`, with one entry per resource. Each entry would give the size of the memory region, a link to the device, and describe any special properties that the memory has. 

In the last minutes of the session, Gorman observed that `hbind()` looks like a combination of the existing `[mmap()](<http://man7.org/linux/man-pages/man2/mmap.2.html>)` and [`set_mempolicy()`](<http://man7.org/linux/man-pages/man2/set_mempolicy.2.html>) system calls; perhaps applications should just use those instead. Hansen added that there will be NUMA nodes for devices providing memory to the system, on x86 systems at least. The new interface essentially makes those NUMA nodes go away, which is suboptimal. Glisse responded that he wants to provide access to memory that is noncoherent or not directly accessible to the CPU; NUMA can't handle either of those. 

Gorman then suggested that [`move_pages()`](<http://man7.org/linux/man-pages/man2/move_pages.2.html>) could be used to place data in the more exotic types of memory; Glisse said that there is no NUMA node to use for that purpose, but the developers pointed out that one could always be added. Or, perhaps, `move_pages()` could grow a new flag to indicate that a device ID is being specified rather than a NUMA node. The final conclusion seemed to be that the `move_pages()` option should be investigated further.
