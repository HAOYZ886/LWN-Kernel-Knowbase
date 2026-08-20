---
title: "What's next for the SLUB allocator"
url: https://lwn.net/Articles/974138/
date: "May 20, 2024"
category: "Memory management-Slab allocators"
author: "By Jonathan Corbet May 20, 2024 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 20, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

There are two fundamental levels of memory allocator in the Linux kernel: the page allocator, which allocates memory in units of pages, and the slab allocator, which allocates arbitrarily-sized chunks that are usually (but not necessarily) smaller than a page. The slab allocator is the one that stands behind commonly used kernel functions like `kmalloc()`. At the [2024 Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), slab maintainer Vlastimil Babka provided an update on recent changes at the slab level and discussed the changes that are yet to come. 

Once upon a time, the kernel contained three slab-allocator implementations. That number had dropped to two in the 6.4 release, when the SLOB allocator (aimed at low-memory systems) was removed. At the [2023 summit](<https://lwn.net/Articles/932201/>), Babka began, the decision had been made to remove SLAB (one of the two general-purpose allocators), leaving only SLUB in the kernel. That removal happened in 6.8. Kernel developers now have greater freedom to improve SLUB without worrying about breaking the others. He thought that nobody was unhappy about this removal, he said, until he saw the recent [report](<https://lwn.net/Articles/970555/>) from the Embedded Open Source Summit, which contained some complaints. Even there, though, the primary complaint seemed to be that the removal had happened too quickly — even though he thought it had taken too long. Nobody seems to be clamoring to have SLAB back, anyway. 

[![\[Vlastimil Babka\]](https://static.lwn.net/images/conf/2024/lsfmm/VlastimilBabka-sm.png)](<https://lwn.net/Articles/974172/>) Last year, some concerns had been expressed that SLUB was slower than SLAB for some workloads. But now, nobody is working on addressing any remaining problems. David Rientjes said that Google is still working on transitioning to SLUB; in the process it has turned up that using SLUB resolves some jitter problems that had been observed with SLAB, so folks there are happy with the change. 

Babka said that he has been working on reducing the overhead created by the accounting of kernel memory allocations in control groups; this cost shows up in microbenchmarks, and "Linus is unhappy" about it. There are some improvements that are ready to go into 6.10, but there is more work to do. Another area of slab development is [heap-spraying defense](<https://lwn.net/Articles/965837/>); these patches are a bit of a problem for him. He can review them as memory-management changes, but he lacks the expertise to judge the security aspect. 

Work is being done on object caching with prefilling. This feature would maintain a per-CPU array of objects that users could opt into; they would be able to prefill (preallocate) the objects prior to allocation so that they are ready to go when needed. That would be useful for objects allocated in critical sections, for example. The initial intended user is the [maple tree data structure](<https://lwn.net/Articles/845507/>), which is currently bulk-allocating a worst-case number of objects before entering critical sections, then returning the unused objects afterward. The object cache would eliminate that back-and-forth while ensuring that objects could be allocated when needed. 

Michal Hocko pointed out that the real problem that is driving this feature is the combination of `GFP_ATOMIC` allocations with the `__GFP_NOFAIL` flag; that combination is difficult for the kernel to satisfy if memory is tight. The allocator currently emits a warning when it sees that combination; avoidance of it on the part of developers would be appreciated, he said. The prefilled object cache is one way of doing that. In the future, some sort of reservation mechanism may be added for such situations as well. 

Another problem exposed by the maple tree has to do with its practice of freeing objects with [`kfree_rcu()`](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/rcupdate.h#L966>) — an approach taken often in kernel code. The problem is that memory freed in this way is not immediately made available for other uses; it must wait for an RCU grace period to pass first. That can lead to an overflow of the per-CPU arrays used by `kfree_rcu()`, causing flushing and, perhaps, a quick refill starting the cycle all over again. To complicate the issue on Android, RCU callbacks are only run on some CPUs, which isn't useful for processing the per-CPU arrays on the CPUs that don't run them. 

The plan is to create a `kfree_rcu()` variant that puts objects in an array and sets them aside to be freed as a whole. Once that has happened, the entire array can be put back into the pool and made available to all CPUs. This array is to be called a "sheaf"; it will be stored in a per-node "barn". One potential problem is that it may become necessary to allocate a new sheaf while freeing objects; allocations in the freeing path need to be avoided whenever possible. The group talked about alternatives for a while without coming to any conclusions. 

Meanwhile, Babka is not satisfied with removing just SLOB and SLAB; next on the target list is [the special allocator](<https://lwn.net/Articles/899274/>) used by the BPF subsystem. This allocator is intended to succeed in any calling context, including in non-maskable interrupts (NMIs). BPF maintainer Alexei Starovoitov is evidently in favor of this removal if SLUB is able to handle the same use cases. The BPF allocator currently adds an [`llist_node`](<https://elixir.bootlin.com/linux/v6.9.1/source/include/linux/llist.h#L60>) structure to allocated objects, making them larger; switching to SLUB would eliminate that overhead. It would also serve to make SLUB NMI-safe and remove the need to maintain yet another allocator. 

Babka would also like to integrate the [objpool allocator](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b4edb8d2d464>), which was added to the 6.7 kernel without any consultation with the memory-management developers at all. Finally, as the session ran out of time, Babka mentioned the possibility of eventually integrating the mempool subsystem (which is another way of preallocating objects). The SLUB allocator could set aside objects for all of the mempools in the system, reducing the overhead as a whole. That, though, looks like a topic for discussion at the 2025 summit.
