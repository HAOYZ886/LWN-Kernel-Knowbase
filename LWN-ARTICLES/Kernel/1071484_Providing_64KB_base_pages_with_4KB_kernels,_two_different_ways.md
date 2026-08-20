---
title: "Providing 64KB base pages with 4KB kernels, two different ways"
url: https://lwn.net/Articles/1071484/
date: "May 11, 2026"
category: "Memory management-Scalability"
author: "By Jonathan Corbet May 11, 2026 LSFMM+BPF"
---

By **Jonathan Corbet**  
May 11, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

Some CPU architectures are able to run with a number of different base-page sizes; using a larger size can often result in better performance at the cost of increased memory use. Other architectures are more limited. At the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), two sessions in the memory-management track explored options for letting processes run with 64KB page sizes when the underlying kernel does not. The first was focused on letting each process have its own page size, while the second concerned bringing 64KB pages to x86 systems. 

#### Per-process page sizes

Using 64KB pages improves performance, but doing so can also create internal fragmentation and a significant amount of wasted memory. That memory-use price tends to limit the use of larger base-page sizes. Ryan Roberts and Dev Jain (remotely) presented a plan to enable the running of processes with page sizes that differ from that of the system as a whole, in an attempt to get the best of both worlds. 

[![\[Ryan Roberts\]](https://static.lwn.net/images/conf/2026/lsfmm/RyanRoberts-sm.png)](<https://lwn.net/Articles/1071711/>) Roberts started by saying that there is a performance gap between systems with larger and smaller page sizes. With a ""random selection of benchmarks"", a performance improvement of 2-17% can be had with a larger page size. But the associated memory usage increase pushes people to stay with the the standard 4KB page size supported by many architectures. The contiguous-PTE support found in some recent processors (where physically contiguous pages can share a translation lookaside buffer (TLB) entry) helps a bit, but even using that feature, the performance gap remains. 

There are a number of reasons for the performance difference. On the software side, a larger page size equates to fewer page faults and shorter least-recently-used (LRU) lists in the kernel. On the hardware side, larger pages lead to better TLB use; a system running with 64KB pages can cover 16 times the memory area with the TLB. Arm CPUs can cache the results of the last page-table walk, speeding the translations of addresses that land within the same page-table entry (PTE) page; larger page sizes increase the coverage of that cache. Using larger pages also just makes the page tables more compact, reducing their TLB and cache impact. 

There is architecture-level work aimed at closing the performance gap, Roberts said, but the results of that work will not be available for some years yet. So there is reason to explore what can be done on the software side instead. One possibility is to give each process its own page size, so that processes that benefit from larger pages can have them without imposing higher memory use on the system as a whole. The Arm architecture, in particular, supports this concept, allowing the kernel to remain with a 4KB page size while letting individual processes run with larger pages. 

Jain took over to describe the proposed implementation, which is split into three layers. The first of those, the "ABI adaptor", is designed to hide the difference between the kernel's page size and that of any given process. Each process's page size is stored in the [`mm_struct`](<https://elixir.bootlin.com/linux/v7.0.1/source/include/linux/mm_types.h#L1123>) structure; it is preserved when a process forks, but may be changed by an [`execve()`](<https://man7.org/linux/man-pages/man2/execve.2.html>) call. Various system calls ([`mmap()`](<https://man7.org/linux/man-pages/man2/mmap.2.html>), for example) will modify length and alignment parameters to match the kernel's page size. That work is fairly straightforward, Jain said, but [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) calls can require more care. The ELF loader is enhanced to understand the alignment needs of processes with different page sizes. There is a fair amount of trickery added to the implementation of various `/proc` files so that a process running with 64KB pages sees the results that would come from a 64KB kernel. 

The second layer is a set of modifications to the kernel's memory-management subsystem. It turns out that a lot of the code paths used to implement transparent huge pages can be reused to provide 64KB pages, on a 4KB kernel, to processes using the larger page size. For such processes, allocation requests specify the page size as the minimum acceptable allocation size; larger pages, up to the PMD-level huge-page size, remain possible. 

The page cache presents challenges of its own, since it is shared by all processes in the system. One option would be to just use 64KB folios there all the time, but that would waste quite a bit of memory when caching small files. So the page cache still uses 4KB pages most of the time. Should a 64KB process map a file with `mmap()`, all 4KB folios from that file will be dropped from the page cache, and any new folios will subsequently be added to the cache at the larger size. 

Kiryl Shutsemau asked whether all filesystems support larger folios in the page cache now; Matthew Wilcox answered in the negative, saying that some filesystems are ""lazy slackers"" that have not yet added that support. The biggest problem, he said, is Btrfs. Wilcox suggested that, as an alternative to dropping page-cache entries, the kernel could go ahead and use 64KB folios as long as they do not extend past the end of the file. 

Lorenzo Stoakes said that this work looks rather invasive, and wondered why it was not possible to just make greater use of multi-size transparent huge pages (mTHPs), which can provide a number of the same benefits. Roberts answered that mTHPs do not provide all of the hardware-level benefits that a larger page size does. Stoakes also worried that extensive use of larger page sizes could put a lot of pressure on the memory-management subsystem's compaction code. 

Time was running short, so Roberts skipped over some of the intended discussion (including the third layer, which is the architecture-specific code that handles differently sized page tables) and moved directly to a list of open items. The first of those had to do with what happens if the kernel attempts an operation requiring a 4KB page size while running in the context of a 64KB process. One option would be to have the process fall back to 4KB pages; that would provide functional correctness, but would lose performance. The alternative is to fail the operation; this idea _seems_ simpler, but would require sprinkling a lot of page-size checks throughout the kernel, Roberts said. 

User-space ABI compatibility is a challenge; the kernel can pretend to be running with 64KB pages when queried by a 64KB process, but it will never be able to emulate everything. Some `/proc` files, for example, simply cannot hide the fact that the kernel is using 4KB pages. It is also not possible for [`/proc/_PID_ /pagemap`](<https://docs.kernel.org/admin-guide/mm/pagemap.html>) to represent a 4KB process when read by a 64KB process. There are also some system calls and other features ([`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>), for example) that cannot be emulated. 

One way to deal with these problems, Roberts said, would be to "defeature" 64KB processes, limiting their functionality. Processes with different page sizes would be invisible to each other, and processes with page sizes larger than the kernel's would be unable to use features like `userfaultfd()`. Any operation that cannot be properly represented to a 64KB process would simply fail. 

Roberts concluded that saying that, while allowing processes to have different page sizes brings benefits, there are some sticky points as well. Adding this feature would also bring a fair amount of churn to the memory-management subsystem. Those benefits may well be worth the trouble, though. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### A 64KB base-page size for x86

Using larger base pages can be a nice solution for workloads that benefit from them, but there is one little problem: some minor architectures, including x86, do not support running with larger base-page sizes. In the next session, Shutsemau proposed a way to work around this limitation on x86 systems. The idea was met with a certain amount of skepticism by the assembled developers, though. 

[![\[Kiryl Shutsemau\]](https://static.lwn.net/images/conf/2026/lsfmm/KirylShutsemau-sm.png)](<https://lwn.net/Articles/1071712/>) Using 64KB base pages, Shutsemau began, can provide a 1.7% performance improvement on ""a very important workload"" on Arm processors; he would like to bring that speedup to x86 systems as well. Using larger pages would reduce the memory overhead of the system memory map, allow for easy (and performance-improving) TLB coalescing, faster I/O operations, and easier allocations of 1GB huge pages. Doing so, he said, requires splitting the kernel's concept of the system page size in two. 

Currently, the `PAGE_SIZE` macro is used throughout the kernel to represent the hardware's base-page size. Shutsemau would phase that out, in favor of `PTE_SIZE`, which describes the hardware's view of the base-page size, and `PG_SIZE`, which is the size of pages as they are managed within the kernel (and seen by user space). The `PAGE_SIZE` macro would only be defined at all if `PTE_SIZE` and `PG_SIZE` are equal. Page-frame numbers, he said, would always refer to `PTE_SIZE` frames. 

Needless to say, there are a lot of places in the kernel that would have to change to reflect this new view of the world. Creating page-table entries would become more complicated, since the offset within the (`PG_SIZE`) page would have to be taken into account; all of the functions that deal with PTEs would gain a new offset parameter. While the kernel is managing 64KB pages, user space would still see the page size as being 4KB, as always. So there would be no user-space changes required to run successfully on such a system. 

The most challenging part, Shutsemau said, is page-fault handling, since multiple PTEs would have to be mapped for each faulting page. User space would only be held to a 4KB alignment requirement, meaning that virtual memory areas (VMAs) could begin or end in the middle of a 64KB page. The page-fault handler might, as a result, end up only mapping part of a page when a fault happens; in such cases, the unmapped part of the page would simply be wasted. Misaligned pages could also lead to memory waste. 

Wilcox said that copy-on-write (COW) faults would become more expensive on these systems, since they would have to fault in surrounding base pages to fill out a 64KB page. David Hildenbrand, instead, worried about how `userfaultfd()` could be implemented; it might need a new operation to install a single PTE rather than a whole page. 

Hildenbrand also suggested it might be better to just go to a 64KB page size throughout the system; that would make life easier for everybody, he said. That, Shutsemau answered, would really just have the effect of shifting the complexity to the architecture code, which would have to implement the fiction of a larger base-page size and hide the details from the rest of the kernel. Going to a larger base-page size would also break some applications. Hildenbrand was unsympathetic to the latter point, saying that such applications should either be fixed or just be run on 4KB systems. 

Jason Gunthorpe said that there has been a lot of experience with 64KB page sizes on Arm systems. Users tend to push back, he said, because there is always one special application that can only run with 4KB pages. Another participant asked why this complexity was needed when the kernel has support for mTHPs that is getting better over time. Part of the problem with that idea, Shutsemau said, is that not all filesystems support larger folios. Sticking with a small base-page size also makes it harder for the system to allocate larger chunks of memory. 

On the topic of memory waste, Hildenbrand suggested the possibility of creating "negative-order folios" to represent sub-page chunks of memory. The idea of using the slab allocator for sub-page allocations was also suggested, but that would not work in all cases. 

As the session ran low on time, Shutsemau acknowledged that he was not seeing a lot of enthusiasm for his proposal. He asked what the fundamental objections were. Hildenbrand answered that, in current kernels, an order-zero folio is a single page; changing that understanding would entail a significant change in how folios are handled. He asked for a cleaner way, one that requires no ""weird part-of-page interfaces"" to reach the desired objectives. 

Gunthorpe said that the fundamental constraint is that there has to be a way to run old applications that require a 4KB page size. It would be better to find a way to solve that problem, with minimal kernel disturbance, on systems with a larger base-page size. The session closed with Hildenbrand saying that other work in the kernel is addressing many of the motivations behind Shutsemau's proposed changes. Given that, he suggested, 64KB base pages may not be the future; the right path may be better optimizing the operation of systems with 4KB pages.
