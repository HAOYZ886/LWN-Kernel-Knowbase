---
title: "Remote per-CPU page list draining"
url: https://lwn.net/Articles/884448/
date: "February 15, 2022"
category: "Memory management-Page allocator"
author: "By Jonathan Corbet February 15, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
February 15, 2022

Sometimes, a kernel-patch series comes with an exciting, sexy title. Other times, the mailing lists are full of patches with titles like "[remote per-cpu lists drain support](<https://lwn.net/ml/linux-kernel/20220208100750.1189808-1-nsaenzju@redhat.com/>)". For many, the patches associated with that title will be just as dry as the title itself. But, for those who are interested in such things — a group that includes many LWN readers — this short patch series from Nicolas Saenz Julienne gives some insight into just what is required to make the kernel's page allocator as fast — and as robust — as developers can make it. 

#### Per-CPU page lists

As its name would suggest, the page allocator is charged with managing the system's memory in units of whole pages; that distinguishes it from the slab allocators, which usually deal in smaller chunks of memory. Allocation and freeing of memory happens frequently within the kernel, making the page allocator a part of many hot paths. A single system call or device interrupt can result in numerous calls into the page allocator, for example, so that code needs to be fast. At times, memory management has been identified as the bottleneck limiting the performance of other parts of the kernel, despite the efforts that have gone into optimizing it. 

At a high level, the page allocator is based on a "buddy allocator", which deals with memory in power-of-two-sized blocks of pages. Among other things, the buddy allocator is good at coalescing adjacent pages into larger blocks when possible. This abstraction begins to be problematic, though, when faced with the needs of contemporary systems, where even a phone handset can have numerous CPUs in it. Maintaining a global buddy structure means a lot of concurrent access to its data; that, in turn, implies locking and cache misses, both of which can wreck performance. 

One of the best ways to mitigate performance problems resulting from concurrent access to shared data is to stop performing concurrent access to shared data. To the extent that each CPU can work within its own private sandbox, without contending with other CPUs, performance will be improved. The page allocator, like many parts of the kernel, uses this approach by keeping per-CPU lists of free pages. 

Specifically, the memory-management subsystem keeps a per-CPU list of free pages in the [`zone` structure](<https://elixir.bootlin.com/linux/v5.16.8/source/include/linux/mmzone.h#L498>) used to describe a memory-management zone. While the reality is (of course) a little more complicated, this structure can indeed be thought of as a simple array of lists of pages, one list for each CPU in the system. Whenever a given CPU needs to allocate a page, it looks first in its per-CPU list and grabs a page from there if one is available. When that CPU frees a page, it puts it back into the per-CPU list. In this way, many page-allocator requests can be satisfied without write access to any global data structures at all, speeding things considerably. Rapid reuse of pages that are cache-hot on the local CPU also helps. 

That only works, of course, if the per-CPU lists have a reasonable number of pages in them. If a CPU needs a page and finds its per-CPU list empty, it will have to take the slower path to obtain memory from the global free list, possibly contending with other CPUs in the process. If, instead, the per-CPU list grows too long, it could tie up memory that is needed elsewhere and some of those pages will need to be given back to the global allocator. Much of the time, though, each CPU can work with its own lists and everybody is happy. 

There is another situation that arises, though, when the system as a whole comes under memory pressure. If the memory-management subsystem reaches a point where it is scrounging for pages anywhere it can find them, it will soon turn an eye to the per-CPU lists, which may contain memory that is sitting idle and ripe for the taking. Unfortunately, even ripe memory cannot just be taken haphazardly; the per-CPU lists only work as long as each CPU has exclusive access to its own lists. If some other CPU pokes its fingers in, the whole system could go up in flames. 

So what is to be done when the system needs to pillage the per-CPU lists for free pages? What happens now is that the kernel asks each CPU to free ("drain") pages out of its own per-CPU lists; this is done with a special, per-CPU workqueue, which is used to queue a callback to free the pages. The workqueue entry will run as soon as each target CPU gets around to scheduling it, which should normally happen fairly quickly. 

This solution is not perfect, though. At best, it causes context switches on each target CPU to run the list-draining callback. But if a target CPU is running in the tickless mode, or if it is running a high-priority realtime task, then the workqueue entry may not run at all for a long time. So any free pages on that CPU will remain locked up in its local lists and, as luck would have it, that's probably where most of the free pages have ended up. 

#### Draining the lists remotely

The patch set is designed to mitigate this problem by making it possible to remotely (i.e. from a different CPU) take pages from a CPU's local lists. A [previous attempt](<https://lwn.net/ml/linux-kernel/20211103170512.2745765-1-nsaenzju@redhat.com/>) added spinlocks to control access to the per-CPU lists, essentially taking away much of their per-CPUness; this solution worked, but it added just the sort of overhead that the per-CPU lists were created to avoid. So those patches did not make it into the kernel. 

The current series, instead, falls back on one of the kernel community's favorite tools for dealing with scalability problems: read-copy-update (RCU). That, in turn, requires the original trick of computer science: adding a layer of indirection. With this patch series, each CPU now has _two_ sets of lists to hold free pages, one of which is in use at any given time, while the other is kept in reserve (and is empty). A new pointer added to the `zone` structure points to the set of lists that is currently in use; whenever a CPU needs to access its local lists, it must follow that pointer to get to them. 

When the time comes to raid a CPU's lists, the raiding CPU will use an atomic compare-and-swap operation to switch the target CPU's pointer to the second (empty) set of lists. The target CPU might still be working with the previous set of lists, though, even after the switch is done, so the raiding CPU must wait for an RCU grace period to pass before actually accessing the old lists. Since this is a per-CPU data structure, the target CPU cannot still hold a reference to the old list once the grace period has passed; at that point, the old lists are fair game and can be emptied out. The target CPU, meanwhile, continues merrily along, though without its local stash of free pages, without ever having been interrupted. 

This approach is not entirely free of performance costs either; adding an extra pointer dereference into the memory-allocation hot paths will add some overhead. Various benchmark results show little difference in most cases, and a 1-3% performance loss in some; the cover letter describes this cost as being acceptable. 

Whether other memory-management developers will agree with that assessment remains to be seen. Kernel developers will work long and hard for a 1% performance increase; they may not be happy to give up that much performance for the benefit of a subset of use cases. In the end, though, the problem being solved is real, and it is not clear that a better solution is on offer. Exciting or not, remote per-CPU list draining may be a feature of future kernels.
