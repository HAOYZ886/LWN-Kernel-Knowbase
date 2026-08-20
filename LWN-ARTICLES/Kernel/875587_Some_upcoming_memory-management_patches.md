---
title: "Some upcoming memory-management patches"
url: https://lwn.net/Articles/875587/
date: "November 12, 2021"
category: "Memory management-GFP flags; Memory management-NUMA systems; Memory management-Scalability; vmalloc"
author: "By Jonathan Corbet November 12, 2021"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 12, 2021

The memory-management subsystem remains one of the most complex parts of the kernel, with an ongoing reliance on various heuristics for performance. It is thus not surprising that developers continue to try to improve its functionality. A number of memory-management patches are currently in circulation; read on for a look at the freeing of page-table pages, `kvmalloc()` flags, memory clearing, and NUMA "home nodes". 

#### Freeing page-table pages

When user space allocates memory, the kernel, obviously, must find pages to satisfy that allocation. But it must also allocate page-table pages to handle the mapping of addresses to the newly allocated memory. For a system with 4KB pages and 64-bit addresses, one page-table page is needed for every 512 ordinary pages of memory (assuming huge pages are not in use). For applications with massive address spaces, the amount of memory used just for page-table pages can be significant. 

That memory can end up being wasted if the user-space pages are reclaimed, perhaps in response to an [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) call. Those pages will be removed from the working set, but the page-table pages that mapped them will remain allocated. If all of the pages mapped by a given page-table page have been reclaimed, the page-table page itself will be empty and serving no purpose. Applications that allocate, then free, large ranges of memory can accumulate a lot of these useless page-table pages, which is less than optimal. 

[This patch set](<https://lwn.net/ml/linux-kernel/20211110105428.32458-1-zhengqi.arch@bytedance.com/>) from Qi Zheng aims to fix that problem. It works by adding a reference count for page-table pages (via yet another overloaded field in `struct page`). Users of the page, such as page-fault handlers, will increment that reference count, ensuring that the page stays around while they do their work. Adding a page-table entry for a user-space page will also increase the reference count, while reclaiming a page will cause the reference count to be decremented. If the reference count drops to zero, the kernel knows that the page-table page does not actually contain any page-table entries and can be freed. 

Test results included with the patch set show that, indeed, reclaiming empty page-table pages can free up a lot of memory for other uses. On the other hand, there is an impact on the page-fault handlers that makes itself felt in a couple of ways. One, of course, is the overhead of maintaining the reference counts as page-table entries are added. But there is also a cost to freeing a page-table page, then having to allocate a new one should that part of the address space become populated again. Overall, the result is an approximately 1% performance hit in page-fault handling. 

That cost may be more than some users want to bear. For now, though, there is no associated knob to turn this behavior off; if this patch set is merged, all systems will free empty page-table pages. Chances are, in any situation where large numbers of page-table pages are affected, performance gain from freeing all of that memory will exceed the page-fault costs, but that may not hold for other types of applications. 

#### More flags in `vmalloc()`

Kernel memory allocated with `vmalloc()` must be mapped into a special part of the kernel's address space. Unlike memory from `kmalloc()` or the page allocator, it is not accessed directly via the kernel's direct memory mapping. Use of `vmalloc()` was, at one time, discouraged; the address space available in the `vmalloc()` area was small, and there is an extra cost to creating the additional mappings. Over time, though, use of `vmalloc()` has grown. The advent of 64-bit systems has eliminated the address-space limitation, and there are increasing numbers of places in the code where multi-page allocations are needed. The chances of successfully allocating a multi-page buffer are much higher if the pages involved need not be located in physically contiguous memory. 

The `vmalloc()` interface, however, has never supported the various `GFP_*` flags passed to `kmalloc()` to influence how the memory is allocated; this limitation persists in add-on functions like `kvmalloc()`, which attempts a `kmalloc()` call with a fallback to `vmalloc()` on failure. This has proved to be a real problem for some kernel subsystems, especially filesystems, that need to be able to allocate memory with flags like `GFP_NOFS`, `GFP_NOIO`, or `GFP_NOFAIL`. As a result, some filesystems have avoided `kvmalloc()`, while others, such as Ceph, have rolled their own memory-allocation functions to work around the problem. 

Michal Hocko has [addressed this problem](<https://lwn.net/ml/linux-kernel/20211025150223.13621-5-mhocko@kernel.org/>) with a patch set adding support for the above `GFP_` flags to the `vmalloc()` subsystem, and to `kvmalloc()` specifically. That makes these functions useful in filesystem settings, and allows the removal of Ceph's special allocation function. As of this writing, one of the precursor patches from that set has made it to the mainline, but the rest have not yet been merged. That may well change before the end of the 5.16 merge window, stay tuned. 

#### Uncached memory clearing

Modern computers make heavy use of memory caches for one simple reason: caches improve performance. So it is interesting to see [this patch set](<https://lwn.net/ml/linux-kernel/20211020170305.376118-1-ankur.a.arora@oracle.com/>) from Ankur Arora that claims to improve memory performance by _bypassing_ caching. As one might expect, this is an improvement that only works in specific circumstances. 

If the kernel needs to zero out a single page of memory, a series of normal, cached writes will almost certainly be the way to go. That allows the cached writes to be flushed to main memory at the system's convenience. A newly zeroed page is also fairly likely to have other data written to it in short order; having that page in cache will speed those writes, and may eliminate the need to write the initial zeroes out to memory at all. Caching is, thus, a performance win here. 

The situation changes, though, when large amounts of memory need to be cleared. If the amount of memory to clear exceeds the size of the last-level cache, it turns out to be faster to just write directly to memory rather than wait for all of those zeroes to be flushed out of the cache. Such a large write will also flush everything else out of the cache, and some of that data is likely to be wanted in the near future. So, for large clearing operations, bypassing the cache seems like the better way to go. 

Arora's patch set thus changes the kernel to use uncached writes whenever a huge (2MB) or gigantic (1GB) page is to be cleared. This kind of operation happens frequently in systems running virtualized guests; a new guest starts off with a range of zeroed memory hosted, when possible, on huge or gigantic pages. Test results included with the patch set show performance improvements of 1.6x to 2.7x for virtual-machine creation. That would seem to be good enough to justify making this change. 

#### Setting a home NUMA node

NUMA systems are characterized by the fact that memory located on the local NUMA node (or a nearby node) is faster to access than memory on a remote node. That means there can be considerable scope for performance improvements by controlling which nodes are used for memory allocations. The kernel provides a number of ways of controlling these allocations, including the recently added `MPOL_PREFERRED_MANY`, which was [covered here](<https://lwn.net/Articles/862707/>) in July. 

Aneesh Kumar K.V. would like to [add another one](<https://lwn.net/ml/linux-mm/20211101050206.549050-1-aneesh.kumar@linux.ibm.com/>), though, in the form of a new system call: 

```
int set_mempolicy_home_node(unsigned long start, unsigned long len,
        				unsigned long home_node, unsigned long flags);
```

This system call will set the "home node" for the `len` bytes of address space beginning at `start` to the NUMA node number passed in `home_node`. The `flags` argument is currently unused. 

The home node is meant to be used in combination with the `MPOL_PREFERRED_MANY` or `MPOL_BIND` memory-allocation policies. Those policies can specify a set of nodes that are to be used for new memory allocations, but do not say anything about which of those nodes, if any, is the preferred one. If a home node has been set with `set_mempolicy_home_node()`, allocations will happen on that node if possible; failing that, the kernel will fall back to one of the other nodes allowed by the in-force policy, preferring the node that is closest to the home node. 

The intent is to give applications a bit more control over memory allocations while avoiding memory from slow nodes. No word yet on when the NUMA developers will throw in the towel and just have user space provide a BPF program to direct memory-allocation policies.
