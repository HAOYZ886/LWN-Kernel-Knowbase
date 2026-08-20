---
title: "A pair of memory-allocation improvements in 5.13"
url: https://lwn.net/Articles/855226/
date: "May 6, 2021"
category: "Memory management-Scalability; vmalloc"
author: "By Jonathan Corbet May 6, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 6, 2021

Among the many changes merged for 5.13 can be found performance improvements throughout the kernel. This work does not always stand out the way that new features do, but it is vitally important for the future of the kernel overall. In the memory-management area, a couple of long-running patch sets have finally made it into the mainline; these provide a bulk page-allocation interface and huge-page mappings in the `vmalloc()` area. Both of these changes should make things faster, at least for some workloads. 

#### Batch page allocation

The kernel's memory-allocation functions have long been optimized for performance and scalability, but there are situations where that work still has not achieved the desired results. One of those is high-speed networking. Back in 2016, networking developer Jesper Dangaard Brouer [described the challenges](<https://lwn.net/Articles/684616/>) that come with the fastest network links; when the system is processing tens of millions of packets per second, the time available to deal with any given packet is highly constrained. The kernel may only have a few hundred CPU cycles available to process each packet, and obtaining a page from the memory allocator may, by itself, require more than that. Using the entire CPU-time budget to allocate memory is not the way to get the best network performance. 

At the time, Brouer asked for an API that would allow numerous pages to be allocated with a single call, hopefully with a much lower per-page cost. The networking code could then grab a pile of memory, and quickly hand out pages as needed. Nobody objected to the request at the time; it is well understood that batching operations can increase throughput in situations like this. But it took some time for that interface to come around. 

Mel Gorman took on that task and put together a patch series, [the sixth version of which](<https://lwn.net/ml/linux-kernel/20210325114228.27719-1-mgorman@techsingularity.net/>) was posted and taken into the -mm tree in March. It adds two new interfaces for the allocation of single ("order-0") pages, starting with: 

```
unsigned long alloc_pages_bulk(gfp_t gfp, unsigned long nr_pages,
        				   struct list_head *list);
```

The allocation flags to use are stored in `gfp`, `nr_pages` is the number of pages the caller would like to allocate, and `list` is a list onto which the allocated pages are to be put. The return value will be the number of pages actually allocated, which could be less than `nr_pages` for any of a number of reasons. The `page` structures for the allocated pages are assembled into a list (using the `lru` entry) and attached to the provided `list`. 

Returning the pages in a linked list may seem a little strange, especially since "linked lists" and "scalability" tend not to go together well. The advantage of this approach is that it does not require allocating any memory to track the allocated pages. Since the list is unlikely to be traversed (there is never a need to walk through the list as a whole), the scalability issues do not apply here. Still, this interface may seem awkward to some. For those who would rather supply an array to be filled with pointers, a different interface is available: 

```
unsigned long alloc_pages_bulk_array(gfp_t gfp, unsigned long nr_pages,
        					 struct page **page_array);
```

This function will store pointers to the `page` structures for the allocated pages into `page_array`, which should really be at least `nr_pages` elements long or unpleasant side effects may appear. Interestingly, pages will only be allocated for NULL entries in `page_array`, so `alloc_pages_bulk_array()` can be used to refill a partially emptied array of pages. This array, thus, must be zeroed before the first call to `alloc_pages_bulk_array()`. 

For users needing more control, the function under the hood that does the work of both `alloc_pages_bulk()` and `alloc_pages_bulk_array()` is: 

```
unsigned int __alloc_pages_bulk(gfp_t gfp, int preferred_nid,
    				    nodemask_t *nodemask, int nr_pages,
    				    struct list_head *page_list,
    				    struct page **page_array);
```

The additional parameters control the location of the allocated pages on a NUMA system; `preferred_nid` is the node to be used if possible, while `nodemask`, if present, indicates the allowable set of nodes. Exactly one of `page_list` and `page_array` should be non-NULL and will be used to return the allocated pages. If both are supplied, `page_array` will be used and `page_list` will be ignored. 

Benchmarks included with the patch set show a nearly 13% speed increase for the high-speed networking case, and something closer to 500% for a Sun RPC test case. Gorman noted, though, that: ""Both potential users in this series are corner cases (NFS and high-speed networks) so it is unlikely that most users will see any benefit in the short term."" The [Sun RPC](<https://git.kernel.org/linus/f6e70aab9dfe>) and [networking](<https://git.kernel.org/linus/be5dba25b4b2>) uses have gone directly into 5.13; others are likely to follow. 

#### Huge-page vmalloc()

Most kernel memory-allocation functions return pointers to either pages or addresses in the kernel's address map; either way, the addresses correspond to the physical address of the memory that has been allocated. That works well for small allocations (one page or below), but physical memory allocations become harder to satisfy as the size of the allocation increases due to the fragmentation of memory over time. For this reason, much work has been done over the years to avoid the need for multi-page allocations whenever possible. 

Sometimes, though, only a large, contiguous region will do; the `vmalloc()` interface exists to serve that need. The pages allocated by `vmalloc()` will (probably) be scattered around physical memory, but they will be made virtually contiguous by mapping them into a special part of the kernel's address space. Traditionally, excessive use of `vmalloc()` was discouraged due to the costs of setting up the mappings and the small size of the dedicated address space on 32-bit systems. The address-space limitation is not a problem on 64-bit systems, though, and use of `vmalloc()` has been growing over time. 

Addresses in the `vmalloc()` range are slower to use than addresses in the kernel's direct mapping, though, because the latter are mapped using huge pages whenever possible. That reduces pressure on the CPU's translation lookaside buffer (TLB), which is used to avoid resolving virtual addresses through the page tables. Mappings in the `vmalloc()` range use small ("base") pages, which are harder on the TLB. 

As of 5.13, though, `vmalloc()` can use huge pages for suitably large allocations thanks to [this patch](<https://git.kernel.org/linus/121e6f3258fe>) from Nicholas Piggin. For `vmalloc()` allocations that are larger than the smallest huge-page size, an attempt will be made to use huge pages rather than base pages. That can improve performance significantly for some kernel data structures, as Piggin [described](<https://lwn.net/ml/linux-kernel/1616036421.amjz2efujj.astroid@bobo.none/>): 

> Several of the most [used] structures in the kernel (e.g., vfs and network hash tables) are allocated with vmalloc on NUMA machines, in order to distribute access bandwidth over the machine. Mapping these with larger pages can improve TLB usage significantly, for example this reduces TLB misses by nearly 30x on a `git diff` workload on a 2-node POWER9 (59,800 -> 2,100) and reduces CPU cycles by 0.54%, due to vfs hashes being allocated with 2MB pages. 

There are some potential disadvantages, including wasting larger amounts of memory due to internal fragmentation; a 3MB allocation may be placed into two 2MB huge pages, for example, leaving 1MB of unused memory at the end. It is also possible that the distribution of memory across NUMA systems may be less balanced when larger pages are used. Some `vmalloc()` callers may be unprepared for huge-page allocations, so they are not done everywhere; in particular, the module loader, which uses `vmalloc()` and could probably benefit from huge pages, does not currently use them. 

Still, the advantages of using huge pages for `vmalloc()` would appear to outweigh the disadvantages, at least in the testing that has been done so far. There is a new command-line parameter, `nohugevmalloc=`, which can be used to disable this behavior if need be. 

Most users are unlikely to notice any amazing speed improvements resulting from these changes. But they are an important part of the ongoing effort to optimize the kernel's behavior wherever possible; a long list of changes like this is the reason why Linux performs as well as it does.
