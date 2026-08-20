---
title: Making swapping scalable
url: https://lwn.net/Articles/704478/
date: "October 26, 2016"
category: "Memory management-Swapping"
author: "By Jonathan Corbet October 26, 2016"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 26, 2016

The swap subsystem is where anonymous pages (those containing program data not backed by files in the filesystem) go when memory pressure forces them out of RAM. A widely held view says that swapping is almost always bad news; by the time a Linux system gets to the point where it is swapping out anonymous pages, the performance battle has already been lost. So it is not at all uncommon to see Linux systems configured with no swap space at all. Whether the relatively poor performance of swapping is a cause or an effect of that attitude is a matter for debate. What is becoming clearer, though, is that the case for using swapping [is getting stronger](<https://lwn.net/Articles/690079/>), so there is value in making swapping faster. 

Swapping is becoming more attractive as the performance of storage devices — solid-state storage devices (SSDs) in particular — increases. Not too long ago, moving a page to or from a storage device was an incredibly slow operation, taking several orders of magnitude more time than a direct memory access. The advent of persistent-memory devices has changed that ratio, to the point where storage speeds are approaching main-memory speeds. At the same time, the growth of cloud computing gives providers a stronger incentive to overcommit the main memory on their systems. If swapping can be made fast enough, the performance penalty for overcommitting memory becomes insignificant, leading to better utilization of the system as a whole. 

As Tim Chen noted in a recently posted [patch set](<https://lwn.net/Articles/704359/>), the kernel currently imposes a significant overhead on page faults that must retrieve a page from swap. The patch set addresses that problem by increasing the scalability of the swap subsystem in a few ways. 

In current kernels, a swap device (a dedicated partition or a special file within a filesystem) is represented by a `swap_info_struct` structure. Among the many fields of that structure is `swap_map`, a pointer to a byte array, where each byte contains the reference count for a page stored on the swap device. The structure looks vaguely like this: 

> ![\[Swap file data structures\]](https://static.lwn.net/images/2016/swap_cluster.svg)

Some of the swap code is quite old; a fair amount dates back to the beginning of the Git era. In the early days, the kernel would attempt to concentrate swap-file usage toward the beginning of the device — the left end of the `swap_map` array shown above. When one is swapping to rotating storage, this approach makes sense; keeping data in the swap device together should minimize the amount of seeking required to access it. It works rather less well on solid-state devices, for a couple of reasons: (1) there is no seek delay on such devices, and (2) the wear-leveling requirements of SSDs are better met by spreading the traffic across the device. 

In an attempt to perform better on SSDs, the swap code was changed in 2013 for the 3.12 release. When the swap subsystem knows that it is working with an SSD, it divides the device into clusters, as shown below: 

> ![\[Swap file data structures\]](https://static.lwn.net/images/2016/swap_cluster1.svg)

The `percpu_cluster` pointer points to a different cluster for each CPU on the system. With this arrangement, each CPU can allocate pages from the swap device from within its own cluster, with the result that those allocations are spread across the device. In theory, this approach is also more scalable, except that, in current kernels, much of the scalability potential has not yet been achieved. 

The problem, as is so often the case, has to do with locking. CPUs do not have exclusive access to any given cluster (even the one indicated by `percpu_cluster`), so they must acquire the `lock` spinlock in the `swap_info_struct` structure before any changes can be made. There are typically not many swap devices on any given system — there is often only one — so, when swapping is heavy, that spinlock is heavily contended. 

Spinlock contention is not the path to high scalability; in this case, that contention is not even necessary. Each cluster is independent and can be allocated from without touching the others, so there is no real need to wait on a single global lock. The first order of business in the patch set is thus to add a new lock to each entry in the `cluster_info` array; a single-bit lock is used to minimize the added memory consumption. Now, any given CPU can allocate pages from (or free pages into) its cluster without contending with the others. 

Even so, there is overhead in taking the lock, and there can be cache-line contention when accessing the lock in other CPUs' clusters (as can often happen when pages are freed, since nothing forces them to be conveniently within the freeing CPU's current cluster). To minimize that cost, the patch set adds new interfaces to allocate and free swap pages in batches. Once a CPU has allocated a batch of swap pages, it can use them without even taking the local cluster lock. Freed swap pages are accumulated in a separate cache and returned in batches. Interestingly, freed pages are not reused by the freeing CPU in the hope that freeing them all will help minimize fragmentation of the swap space. 

There is one other contention point that needs to be addressed. Alongside the `swap_info_struct` structure, the swap subsystem maintains an `address_space` structure for each swap device. This structure contains the mapping between pages in memory and their corresponding backing store on the swap device. Changes in swap allocation require updating the [radix tree](<https://lwn.net/Articles/175432/>) in the `address_space` structure, and that radix tree is protected by another lock. Since, once again, there is typically only one swap device in the system, that is another global lock for all CPUs to contend for. 

The solution in this case is a variant on the clustering approach. The `address_space` structure is replicated into many structures, one for each 64MB of swap space. If the swap area is sized at (say) 10GB, the single `address_space` will be split 160 ways, each of which has its own lock. That clearly reduces the scope for contention for any individual lock. The patch also takes care to ensure that the initial allocation of swap clusters puts each CPU into a separate `address_space`, guaranteeing that there will be no contention at the outset (though, once the system has been operating for a while, the swap patterns will become effectively random). 

According to Chen, current kernels add about 15µs of overhead to every page fault that is satisfied by a read from a solid-state swap device. That, he says, is comparable to the amount of time it takes to actually read the data from the device. With the patches applied, that overhead drops to 4µs, a significant improvement. There have been no definitive comments on the patch set as of this writing, but it seems like the sort of improvement that the swap subsystem needs to work well with contemporary storage devices.
