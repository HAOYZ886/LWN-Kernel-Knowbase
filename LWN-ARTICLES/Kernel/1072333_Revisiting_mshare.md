---
title: Revisiting mshare
url: https://lwn.net/Articles/1072333/
date: "May 13, 2026"
category: "Memory management-Page-table sharing"
author: "By Jonathan Corbet May 13, 2026 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 13, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Linux can share memory between processes, but each process (almost always) has its own set of page tables. In situations where vast numbers of processes are sharing a memory region, the combined size of the page tables can exceed that of the shared memory itself. There has, thus, long been an interest in enabling unrelated processes to share page tables referring to shared memory. Anthony Yznaga is the latest developer to try to push this idea (known as "mshare") forward; he described the status of that work in a memory-management-track discussion at the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) (LSFMM+BPF). 

[![\[Anthony Yznaga\]](https://static.lwn.net/images/conf/2026/lsfmm/AnthonyYznaga-sm.png)](<https://lwn.net/Articles/1072355/>) This is not the first (or second) time that page-table sharing has made it onto the LSFMM+BPF agenda; it was most recently [discussed in 2024](<https://lwn.net/Articles/974512/>), when Khalid Aziz updated the group on the proposal. Aziz has since retired, and Yznaga has picked up the work. 

The overall shape of this patch series has not changed; sharing starts when a process creates a shared memory region by creating a file in a special `msharefs` filesystem. That region is created along with its own [`mm_struct` structure](<https://elixir.bootlin.com/linux/v7.0.5/source/include/linux/mm_types.h#L1123>), which is used to manage the page tables for the region. Each sharing process can then attach to this region by opening and mapping the `msharefs` file, resulting in the creation of a special virtual memory area (a "window VMA") representing that region in the process's address space. Page faults and other memory-management operations, upon encountering the window VMA, follow the pointer to the special `mm_struct` and operate on the page tables there. 

At least, that is how things looked in 2024, and again in 2025 when Yznaga posted [an updated version of the patch set](<https://lwn.net/ml/linux-mm/20250820010415.699353-1-anthony.yznaga@oracle.com/>). In the session, though, he let it be known that, while the implementation of mshare is substantially the same, the API has switched back to system-call form, as had been the case with earlier versions. Back then, a single `mshare()` system call had been proposed; now there is a whole set of them. The shared region is now created with `mshare_create()`: 

```
int mshare_create(unsigned int flags);
```

This call will return a file descriptor representing the new region; the only supported `flags` value is `O_CLOEXEC`. The size of the region must subsequently be set with an [`ftruncate()`](<https://man7.org/linux/man-pages/man2/truncate.2.html>) call. A call to `mshare_attach()` will map the shared region into the calling process's address space: 

```
int mshare_attach(int fd, unsigned int offset, unsigned int size,
        		      void *addr, unsigned int flags);
```

(Note that Yznaga did not show the types of the parameters on his slide, so I have filled in something plausible). There is an `mshare_map()` to do the equivalent of an [`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>) call, setting up backing store for the addresses within the shared region. There are various other calls, including `mshare_advise()` and `mshare_protect()`, to control the management of this region. Yznaga did not go into details about how other processes find and attach to this region. 

The ownership model of the shared region has changed somewhat, in that the process calling `mshare_create()` is the owner for the life of the process; when that process exits or closes the file descriptor, the region disappears and mappings in other processes are removed. This change, he said, simplifies control-group accounting, makes the lifetime of the region clear, and provides a target for the out-of-memory killer should things reach that point. 

Yznaga concluded with a summary of problems he is working on now. Page-table walking is a big challenge he said, especially getting the locking between the window VMAs and the mshare region's VMA correct. The resident-set-size statistics for mshare-using processes are wrong; the information for the shared region is stored in the special `mm_struct` and not exposed anywhere. The current design requires the process that created the region to stick around; there would be value in some sort of ownership-transfer mechanism so that the creator could hand the region off and exit. 

He is also looking for more potential use cases for this feature. Jason Gunthorpe mentioned high-performance-computing processes that need to share resources, so it would be good to show that this feature can work for that use case. Another participant mentioned [the Android Zygote process](<https://source.android.com/docs/core/runtime/zygote>), which serves as the parent for all apps in the system. There is quite a bit of sharing between those processes, and thus potential benefit from mshare, but any changes made by a process to the region should not be visible to other attached processes, so it would be necessary to unshare page tables in that case. 

Another participant asked how TLB flushing is handled in the shared region. There is, Yznaga answered, a linked list of all processes sharing the region; when a TLB flush happens, that list is traversed and each process is flushed individually. Gunthorpe observed that this list sounded a lot like an MMU notifier; Yznaga said that he had tried using notifiers, but they did not work in that case. 

Matthew Wilcox noted that allowing each process to map the shared region at a different virtual address increases the complexity of the feature; perhaps requiring the region to be mapped at the same address everywhere would be a useful simplification? The response in the room made it clear that this was not a popular idea. The session ended with Wilcox suggesting that, once the feature finally lands, it will be possible to remove the page-table sharing implemented by the [hugetlbfs subsystem](<https://docs.kernel.org/admin-guide/mm/hugetlbpage.html>), which is currently the only way to get this kind of sharing on Linux systems.
