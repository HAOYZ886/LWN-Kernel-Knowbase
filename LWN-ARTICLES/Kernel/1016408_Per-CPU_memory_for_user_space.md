---
title: "Per-CPU memory for user space"
url: https://lwn.net/Articles/1016408/
date: "April 8, 2025"
category: Memfd
author: "By Jonathan Corbet April 8, 2025 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
April 8, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

The kernel makes extensive use of per-CPU data as a way to avoid contention between processors and improve scalability. Using the same technique in user space is harder, though, since there is little control over which CPU a process may be running on at any given time. That hasn't stopped Mathieu Desnoyers from trying, though; in the memory-management track of the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit, he presented a proposal for how user-space per-CPU memory could work. 

[![\[Mathieu
Desnoyers\]](https://static.lwn.net/images/conf/2025/lsfmm/MathieuDesnoyers-sm.png)](<https://lwn.net/Articles/1016414/>) Desnoyers started by saying that his objective is to help user-space developers make better use of [restartable sequences](<https://lwn.net/Articles/883104/>), which facilitate some types of access to per-CPU data by interrupting a process if it is migrated during a critical section. User-space applications generally use thread-local storage for this kind of code, but that becomes inefficient if there are more threads running than CPUs to run on. Thread-local storage must also be defined statically, making it inflexible to work with, and it can slow down thread creation if the area is large. 

So he would like to provide true per-CPU data as an alternative. One way _not_ to do that, he said, is to structure per-CPU data in an array indexed by the CPU number. The implementation would be relatively simple; code could just obtain its current CPU from [`sched_getcpu()`](<https://man7.org/linux/man-pages/man3/sched_getcpu.3.html>) or from the restartable-sequences shared-memory area. But if the array is packed, the result will be cache-line bouncing between CPUs, eliminating much (or all) of the performance benefit. If, instead, array entries are aligned to cache lines, there will likely be a lot of wasted space between them. 

The kernel's per-CPU allocator, he said, maps a range of address space on each CPU to provide access to that CPU's local memory space; allocations just return a single address that can then be used on all CPUs. He has implemented a similar approach in the [librseq](<https://github.com/compudj/librseq>) library. The allocator creates a set of memory pools, one per CPU; allocations then return an offset that is the same on every CPU. It is essentially the cache-line-aligned array approach, but the allocator packs allocations within each CPU's area, reducing the wasted memory between those areas. It can support multiple pools, thus isolating users from each other. 

The kernel's [memfd feature](<https://lwn.net/Articles/593918/>) is used to create the per-CPU memory pool. It is about the only way, he said, that allows him to create the various mappings into the same area that the feature needs. 

There are some potential problems with this approach, though. What if a four-thread process is running on a system with 512 CPUs? Allocating and initializing memory for all of those CPUs would be wasteful, most of it will never be used. So, instead, the library initializes one CPU's worth of memory in a special "initialization area", then creates a copy-on-write mapping of that area for each CPU. Any CPU reading from its area will read from that single copy; if a CPU writes to its area, the page will be copied and will become truly CPU-local. 

Another concern relates to what happens when a process forks; the per-CPU area will be shared across the fork, which may not be what is wanted. He is considering adding a memfd flag (`MFD_PRIVATE`) that would make a memory area per-process; a fork would then result in the child getting a separate copy of that area. For now, he is using an ""inconvenient workaround"" consisting of a couple of [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) operations to detect and handle forks. As part of that, the library maps a special "canary page" that is set to be cleared when a fork happens; the contents of that page can then be checked to detect forks. 

In the future, he is considering adding the ability to allocate variable-sized elements within the per-CPU memory pool. The placement of guard pages between each CPU's area would prevent some cache-line bouncing caused by one CPU prefetching into another CPU's area. He also has thoughts about improving the control-group CPU controller to allow for maximum-concurrency limits, which would make it possible to put tighter limits on the number of entries needed in the pool. 

Desnoyers concluded his presentation by returning to the `MFD_PRIVATE` idea, which he sees as the best way of solving the fork problem. This feature would be useful in other contexts, he said. The [MESH allocator](<https://dl.acm.org/doi/pdf/10.1145/3314221.3314582>) needs this kind of feature, as do Google's dynamic-analysis tools. David Hildenbrand said that `MFD_PRIVATE` could be a reasonable addition, but thought that its use should also imply behavior like that obtained with `MADV_WIPEONFORK`, where the memory is zeroed when a fork happens. Desnoyers answered that this behavior might not be wanted; a child process could still make use of the per-CPU data from the parent, but would have its own copy for any changes it made. 

Suren Baghdasaryan commented on the possible use of guard pages to prevent cross-CPU prefetching, noting that this behavior is architecture-dependent. He wondered if Desnoyers has considered how this work interacts with [cpusets](<https://man7.org/linux/man-pages/man7/cpuset.7.html>). Desnoyers said cpusets and per-CPU memory do work together, but there are some challenges. Since his library will not get CPU-hotplug notifications, it has to be ready for unexpected changes in the CPU topology. 

Hildenbrand asked how processes can be sure that the CPU does not change underneath them while accessing per-CPU data; Desnoyers answered that restartable sequences are the usual way to do that. I followed up, asking whether restartable sequences were the only safe way to work with this memory; he said that there are other options, including atomic operations or [rseq locks](<https://lwn.net/Articles/946870/>). 

The session concluded at that point. Desnoyers has not posted the slides from this session, but [the slides from his February FOSDEM talk](<https://fosdem.org/2025/events/attachments/fosdem-2025-6245-waste-free-per-cpu-userspace-memory-allocation/slides/238510/presentat_egMx2j1.pdf>) cover the same points.
