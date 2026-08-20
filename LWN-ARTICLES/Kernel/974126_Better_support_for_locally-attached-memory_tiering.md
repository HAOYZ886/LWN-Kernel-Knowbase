---
title: "Better support for locally-attached-memory tiering"
url: https://lwn.net/Articles/974126/
date: "May 20, 2024"
category: "Memory management-Tiered-memory systems"
author: "By Jonathan Corbet May 20, 2024 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 20, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The term "memory tiering" refers to the management of memory placement on systems with multiple types of memory, each of which has its own performance characteristics. On such systems, poor placement can lead to significantly worse performance. A memory-management-track discussion at the [2024 Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) took yet another look at tiering challenges with a focus on upcoming technologies that may simplify (or complicate) the picture. 

A quick note: this session mentioned future plans from a number of different companies, and some participants were worried about revealing too much or breaking non-disclosure agreements. For this reason, it was requested that this session be reported without naming the people involved or attributing any statements. Apologies for the forthcoming vagueness, but hopefully the important parts get through. 

The specific focus of this session was obtaining optimized placement on systems with [CXL memory](<https://en.wikipedia.org/wiki/Compute_Express_Link>) attached. This memory is large and flexible in its use (it can, for example, be moved from one server to another in some configurations) but, since it is more distantly attached, it is also slower. If, though, the system can learn to use this memory properly and place the right data there, there is a huge potential for both performance improvements and cost savings. Preferably, this would happen without the need for new, specialized interfaces. The kernel's tiering support should be useful for all systems, and it should be extensible, since memory types will change in the future. The hope for the discussion was to reveal any constraints in the memory-management subsystem that would impede this support and to bring the developers working in this area together. 

#### The trouble with NUMA balancing

The current approach to tiering is based on NUMA balancing — different types of memory appear to be different (CPU-less) NUMA nodes, and the kernel manages the placement of memory on each node. The advantage is that the kernel's NUMA-balancing code is ten years old at this point, and is relatively mature. Tiering support has been added more recently, with a special mode that attempts to optimize memory placement. 

The use of NUMA balancing for tiering is not ideal, though; it is too slow in a number of ways. Perhaps the biggest problem is page promotion. It is relatively easy for the kernel to notice data that is not seeing frequent use and demote it to slower memory. The promotion path — observing frequent use of data on slow memory and moving it to a faster tier — is harder. Promotion needs to be fast; once a process starts using some data, it tends to work on it for a while; if that data is not promoted immediately, performance will suffer. 

NUMA balancing uses a sliding-window technique, where memory is access-protected and the resulting faults (on the pages that are actually accessed) are noted. This algorithm takes time and is not responsive enough for the promotion case; performance will decay while pages are waiting to be promoted. NUMA balancing is also a system-wide task, but it really needs to be job-wide, and should eventually be controlled with memory control groups. An additional challenge in making all of this work is a lack of good benchmarks to measure the effectiveness of tiering algorithms. 

It was pointed out that one type of memory — that which is hosted on peripheral devices like GPUs — is special in its own way. Unmapping that memory (prior to migrating it and mapping it in its new location) can stop the device in its tracks and kill performance, so automatic tiering has to be disabled on such systems. The lack of device awareness in the kernel's tiering mechanisms needs to be fixed. 

One possible approach that was suggested was to focus on DAMON (which was [discussed](<https://lwn.net/Articles/973702/>) earlier that day) as a flexible way to implement tiering algorithms. On the other hand, DAMON also feels a bit like a separate memory-management subsystem, and it could be better to keep this support in the core. 

An upcoming change that should help with this task is that, in the future, CXL controllers will allow the kernel to easily observe which pages are being accessed. That will be a fast source of truth, under the kernel's control. But it is not clear how that information can be used. It seems that either NUMA balancing or DAMON could be extended to take advantage of CXL hot-page detection. One developer said that hot-page detection looks like many hardware-assistance features that promise to help, but where the hardware developers always get it wrong and the problem still has to be solved in software. 

NUMA balancing was designed to converge on an optimal solution for a given workload and not move a lot of data around. That makes it hard to extend to this case, where active migration of data is needed. Trying to create a complex policy that will work for all workloads is impossible, developers said, so it will be necessary to make NUMA balancing more extensible — or to use a different mechanism entirely. It is important, one developer suggested, to avoid conflating the mechanism for detecting hotness for the one that moves pages; the two need to be firmly decoupled. 

#### Possible solutions

Various ideas flew around the room. [A 2023 session](<https://lwn.net/Articles/931812/>) had looked at the use of hardware performance counters for page aging; perhaps that work could be extended here. It seems, though, that not all CPUs have performance-management units that provide the information that is needed. The [multi-generational LRU](<https://lwn.net/Articles/856931/>) already contains several tiers internally that could be used to manage tiering, but one developer said that experiments with LRU-based hot-page detection did not work out as well as had been hoped. It was also said that "hot-page detection" should really be "hot-folio detection", and that scanning should work better in general for larger folios. 

One component of NUMA balancing is often called "workload follows memory"; if a task finds itself running far from its memory, it can be migrated to a closer node. That doesn't work for CXL, though, since CXL nodes have no CPU, so tasks cannot be migrated there. It was suggested that task migration should be disabled in general, that balancing workloads across CPUs is no longer relevant in our world. Task migration can throw NUMA systems out of balance, forcing the migration of memory to follow tasks around. Migration can also split communicating tasks apart from each other. Often, it was said, it is better to just leave the system alone. 

This was, of course, a room full of memory-management developers; had there been CPU-scheduler developers present, that assertion would likely have been challenged. Even in this crowd, one developer disagreed, saying that the problem isn't task migration; instead, the CPU scheduler just isn't being given enough information to make the best decisions. 

There is, it seems, a need for some sort of "hot-memory abstraction" for the kernel. It could take advantage of "accessed" bits in the page tables, performance-monitoring units, the upcoming CXL hot-page detection feature, or any "future hardware innovation" that might be in the works. Whatever information is available should be brought in and provided in a single interface. It could be useful for more than tiering; NUMA balancing would also benefit from better information. One possible problem is that, while tiering does not normally need to know which CPU is accessing data, NUMA balancing depends heavily on that information. 

Toward the end of the session, mechanisms for acting on this information were discussed. One possibility is to push as much of it to user space as possible; the migration of memory will often require changes elsewhere in the system (such as redirecting interrupts) that only user space can know about anyway. The kernel currently provides a [`memory.reclaim` knob](<https://docs.kernel.org/admin-guide/cgroup-v2.html#memory-interface-files>) to trigger reclaim; perhaps a `memory.demote` could be provided as well? Maybe there needs to be a `kpromoted` thread, or perhaps that task should be handled by user space. 

Memory promotion, it was said, should be done asynchronously, unlike how NUMA migration is done. Moving memory synchronously can create latency blips that user space might notice; performing promotion asynchronously would still be noticeable, but it would not stall an application in the same way. 

At the conclusion of the session, it was repeated that a proper solution in this area could lead to "massive amounts" of money being saved, especially in hyperscaler deployments. A memory-tiering working group is being formed to continue work in this area and to ensure that all of the known use cases are handled.
