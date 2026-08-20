---
title: "Persistent memory, with and without page structures"
url: https://lwn.net/Articles/656197/
date: "September 2, 2015"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet September 2, 2015"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
September 2, 2015

Persistent memory offers the prospect of large amounts (e.g. terabytes) of directly attached memory that retains its contents over a system reboot or power cycle. It also offers a number of interesting design problems with regard to how it should be managed; persistent memory looks a lot like ordinary memory, but it differs in a number of important ways. As a result, there has been a long discussion over how to deal with this memory and, in particular, whether the kernel should use `page` structures to describe it or not. As shown in some recent patch sets, the discussion continues to evolve, and it seems to be heading toward an interesting answer to the `struct page` question. 

For those needing a quick recap, [`struct page`](<https://lwn.net/Articles/565097/>) is the kernel's fundamental memory-management data structure; one `page` structure exists for each page of memory present in the system. In current kernels, though, persistent memory does not have accompanying `page` structures for a simple reason: the amount of memory required to hold all of those structures looks prohibitive. Storing them in the persistent memory array itself is possible (and discussed further below), but `page` structures change frequently, making them a poor fit for persistent memory storage, which (1) tends to be slow for writes, and (2) will wear out more quickly if subjected to sustained frequent writes. 

As long as a persistent-memory array is treated like a disk drive, there is no need for `page` structures. But if persistent memory is to take part in DMA or direct I/O operations, it currently needs those structures; for that reason, such operations do not currently work on persistent memory. This problem is widely seen as needing a fix. 

When we [last looked at the discussion](<https://lwn.net/Articles/644079/>) in May, there was a push toward using page-frame numbers (PFNs) as a replacement for `page` structures in various I/O paths. A PFN is easily derived from a page's physical address, so it is an easy and obvious way to refer to a physical page of memory — if the additional information stored in the `page` structure is not needed for any given operation. In May, though, it was becoming clear that this information cannot always be done without, and, thus, that this approach had its limitations, especially when it came to supporting direct I/O, which is the most scalable I/O mode that the kernel offers. 

#### Using page-frame numbers

Nonetheless, work continues on the PFN-based approach. Christoph Hellwig posted [this patch series](<https://lwn.net/Articles/654273/>) adapting the DMA subsystem so that it could manage scatter/gather lists containing PFNs. A scatter/gather list describes an I/O operation that is spread across multiple regions of memory; these lists are used for almost all nontrivial I/O operations, since I/O buffers are rarely situated in a single, physically contiguous block of memory. Making scatter/gather lists work without `page` structures would, for the most part, solve the problem of doing DMA on buffers stored in persistent memory. Christoph's patch doesn't do that, but it abstracts out the references to `page` structures, making it easy to use PFNs instead in the future. 

Beyond this preparatory work, though, the kernel needs the ability to work more extensively with PFNs. Happily, on the same day, Dan Williams posted [a new revision](<https://lwn.net/Articles/654396/>) of his patch series implementing the `__pfn_t` type for the management of pages by PFN. The new `__pfn_t` type is simpler than it was the last time around: 

```
typedef struct {
        	unsigned long val;
        } __pfn_t;
```

There is no more trickery with storing PFNs and `struct page` pointers in the same structure. There are, however, a few bits of `val` that are used for related purposes: to chain entries in scatter/gather lists, for example and, in the case of the `PFN_DEV` bit, to indicate that the PFN has no associated `page` structure in the system. There is a set of helper functions to do things like get the actual PFN number (`__pfn_t_to_pfn()`) or the physical address (`__pfn_t_to_phys()`) associated with a `__pfn_t` value. 

One common use for a `page` structure is to map the associated page into the kernel's address space with `kmap_atomic()`; that allows the kernel to manipulate that page directly. For code dealing with PFN values instead of `page` structures, Dan's patch set adds `kmap_atomic_pfn_t()` to do the same job; it will work regardless of whether the PFN it is given refers to ordinary or persistent memory. Interestingly, when successful, `kmap_atomic_pfn_t()` returns with the RCU read lock held, and `kunmap_atomic_pfn_t()` expects to release that lock. 

The final patch in the series converts the scatter/gather DMA code over to the use of PFNs instead of page structures. That should enable the DMA code to work on persistent memory, though it seems that there may be some remaining issues on a few architectures. 

The PFN-based approach is not universally admired; in particular, there has been some [resistance](<https://lwn.net/Articles/656200/>) from Boaz Harrosh, who believes that `page` structures should always be used with persistent memory — and who posted [a patch set](<https://lwn.net/Articles/608538/>) to that effect one year ago. Boaz's patches don't seem to have been developed since then, though, and his objections do not appear to be slowing things down much. David Miller has also [expressed discomfort](<https://lwn.net/Articles/656201/>) with the idea of memory without `page` structures, for what it's worth. 

#### Back to page structures

These misgivings notwithstanding, persistent-memory developers clearly see value in providing access to this memory without associated `page` structures. So it may have come as a surprise to some when Dan also posted [this patch series](<https://lwn.net/Articles/656263/>) adding none other than `struct page` support for persistent memory. In the end, it seems, there are certain things that simply cannot be done without `page` structures; direct I/O and remote DMA are two features at the top of that list. This patch set allows the creation of these structures on systems where they are needed while allowing the rest to avoid the associated overhead. 

This patch set adds a new type of block device that sits on top of the existing [pmem driver](<https://lwn.net/Articles/640113/>) that was merged for the 4.1 kernel. The driver for this new device will add a persistent-memory range to the system's memory map, using the memory hotplugging mechanism. The memory goes into a special zone (`ZONE_DEVICE`) created for this purpose, though, so it will not be made available to the rest of the system like ordinary memory. As part of this process, the driver allocates an array of `page` structures to describe this memory range. 

Allocating that array brings us back to the problem of memory consumption: a large persistent-memory block will require a large array of `page` structures to describe it. One possible solution to this problem is to introduce a new structure for variably sized pages, or to simply use huge pages, but Dan's patch set sticks to the ordinary `page` structure describing 4KB pages. So the memory-consumption problem remains. 

The original version of the patch offered an interesting approach to that problem: the decision of _where_ these `page` structures should live was pushed out to user space. By tweaking a sysfs attribute, the system administrator could direct those structures into ordinary memory, or could instead cause them to be stored in the persistent-memory array itself. So large arrays could host their own `page` structures. As mentioned above, persistent memory may not be the best place to store those structures, but, for many use cases, it may work well enough, and this approach does make the problem of `page` structures taking up too much RAM go away. 

The [current version of this patch set](<https://lwn.net/Articles/656264/>) drops that feature, though, and instead stores `page` structures in RAM unconditionally. That change simplifies the memory-management changes, making it easier to get the patch set reviewed and merged. Expect the store-in-persistent-memory option to return in the future, though, as the huge arrays we've been promised finally start to show up in the mass market. 

Meanwhile, we now have a set of patches that make persistent memory behave almost entirely like ordinary memory with regard to management within the kernel. That means that, assuming this work is merged, Linux is essentially ready to support the use of persistent memory for a wide variety of use cases. What remains, at this point, is to see just what developers will do once they have terabyte-sized arrays of persistent memory available to play with.
