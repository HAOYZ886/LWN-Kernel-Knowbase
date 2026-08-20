---
title: Direct I/O and DMA for persistent memory
url: https://lwn.net/Articles/672457/
date: "January 20, 2016"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet January 20, 2016"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
January 20, 2016

The last year or so has seen a great deal of work toward improving the kernel's support of persistent-memory (or "nonvolatile-memory") devices. Persistent memory looks like regular memory to the system in a number of ways, but it differs in others, most notably in that its contents persist across reboots and power cycles. The upcoming 4.5 kernel contains some core memory-management changes that address one of the biggest items left on the "to do" list for persistent memory: support for DMA and direct I/O. Getting there was a multi-step process, though. 

One of the biggest areas of disagreement with regard to persistent-memory support has been whether that memory should be represented in the system memory map. Doing so means setting aside considerable amounts of memory for a `page` structure representing each persistent-memory page; with large persistent-memory arrays, those structures could occupy a significant percentage of the system's RAM — or not fit at all. But the lack of `page` structures makes persistent memory invisible to much of the low-level memory-management code and, as a result, rules out operations like direct I/O. Since some of the prominent use cases for persistent memory (serving as a fast cache for a huge disk array, for example) require DMA and direct I/O, this was seen as a significant problem. 

The solution, merged for 4.5, is evolved from the approaches [described here](<https://lwn.net/Articles/656197/>) in September 2015. At that point, there was a significant push to use page-frame numbers (PFNs) as a replacement for `page` structures in much of the memory-management subsystem. If all the memory in the system is seen as a huge array, a PFN is simply an index into that array for a specific page. Any memory that is addressable by the CPU will have an associated PFN, so using the PFN seems like a logical way to refer to that page. There is a catch, though: `struct page`, beyond just identifying a page, also contains [crucial information](<https://lwn.net/Articles/565097/>) about how that page is being used. So it's not possible to do without `struct page` entirely. 

The approach found in the 4.5 kernel, implemented by Dan Williams, starts with some of the PFN-based ideas that have been passed around in the past, but does not stop there. There is a new type to represent a PFN and some associated information: 

```
typedef struct {
    	unsigned long val;
        } pfn_t;
```

Adding this type required renaming a couple of `pfn_t` types already existing in other parts of the kernel. The `val` member contains the actual PFN, but the high-order bits are used to encode a few extra flags. Two of them, `PFN_SG_CHAIN` and `PFN_SG_LAST`, are meant to be used with scatter-gather lists for DMA that use PFNs rather than pointers to `page` structures, but the scatter-gather part has not (yet) been merged, so these flags are unused as of this writing. Beyond that, `PFN_DEV` indicates a page frame stored on special "device" memory that may not have an associated `page` structure, and `PFN_MAP` indicates that a `page` structure _does_ , in fact, exist. 

The kernel has had the ability to (easily) create `page` structures for persistent memory since 4.3, when `devm_memremap_pages()` was introduced by Christoph Hellwig: 

```
/* The v4.3 version of this function */
        void *devm_memremap_pages(struct device *dev, struct resource *res);
```

This function will map the region described by `res` into the kernel's virtual address space, allocating `page` structures for it along the way. It is not a complete solution to the problem, though, for a couple of reasons. One is that it lacks the reference-counting support needed to ensure that a persistent-memory device doesn't disappear while it is in use. The other, of course, is the same old problem: for a huge persistent-memory array, there just isn't room in RAM for all of those `page` structures. 

The lack of reference counting matters for use cases like DMA and direct I/O; it would not do to have some persistent memory (or the mapping to it) disappear in the middle of an operation. In 4.5, this problem is fixed by requiring persistent-memory drivers to provide a [`percpu_ref` structure](<https://lwn.net/Articles/557478/>) to go with any memory array that is mapped into the kernel's address space. A pointer to this reference counter is then stored (with a level of indirection) in the already overloaded `page` structure; since persistent-memory `page` structures will never appear in the memory-management subsystem's LRU lists, the space occupied by the `lru` field is available for this purpose. 

The 4.5 work introduces a new flag, `_PAGE_DEVMAP`, which is stored in the page-table entry itself when persistent memory is mapped into a process's address space. Code that creates references to this memory, `get_user_pages()` for example, will see that flag and respond by incrementing the `percpu_ref` counter associated with the persistent-memory array. As long as that counter remains elevated, it will not be possible to remove the memory from the system. 

The other problem — the size of all those `page` structures — has an obvious solution: store those structures in the persistent-memory array itself. This solution is not ideal; `page` structures can change frequently, which mixes poorly with the relatively high cost of writing to persistent memory. But it is better than having no `page` structures at all. So, in 4.5, drivers for persistent memory can set aside a chunk of each array for the storage of `page` structures. That is done by filling in a `vmem_altmap` structure: 

```
struct vmem_altmap {
    	const unsigned long base_pfn;
    	const unsigned long reserve;
    	unsigned long free;
    	unsigned long align;
    	unsigned long alloc;
        };
```

The `base_pfn` field points to the base of the array. A driver can keep some of the memory for its own purposes by storing the amount in the `reserve` field; the `free` field should be set to the number of pages that can be used to hold `page` structures. A simple allocator built into the memory-management code will then use those pages (tracking them with the `alloc` field) to create `page` structures when mapping the array into kernel space. 

All of these additions come together in the 4.5 version of `devm_memremap_pages()`: 

```
void *devm_memremap_pages(struct device *dev, struct resource *res,
    			      struct percpu_ref *ref, struct vmem_altmap *altmap);
```

With this infrastructure in place, a persistent-memory driver can easily set up an array that is mapped into kernel memory and which has `page` structures behind it. That allows functions like `get_user_pages()` to work, and, as a consequence, direct I/O and DMA also work. An additional benefit (from a bit more work) is that huge-page mappings into persistent memory work properly. 

Without doubt, work on supporting persistent memory will continue for some time; this memory represents a major change in how our systems work. But, as of the 4.5 kernel, it would appear that the important low-level pieces are in place. What remains now is figuring out the best ways to actually use terabytes of directly connected persistent memory, both within the kernel and at the application level. It will be interesting to see what developers come up with in the next few years.
