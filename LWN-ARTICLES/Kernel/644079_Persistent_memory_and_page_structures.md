---
title: Persistent memory and page structures
url: https://lwn.net/Articles/644079/
date: "May 13, 2015"
category: "Memory management-Nonvolatile memory"
author: "By Jonathan Corbet May 13, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
May 13, 2015

As is suggested by its name, persistent memory (or non-volatile memory) is characterized by the persistence of the data stored in it. But that term could just as well be applied to the discussions surrounding it; persistent memory raises a number of interesting development problems that will take a while to work out yet. One of the key points of discussion at the moment is whether persistent memory should, like ordinary RAM, be represented by `page` structures and, if so, how those structures should be managed. 

One [`page` structure](<https://lwn.net/Articles/565097/>) exists for each page of (non-persistent) physical memory in the system. It tracks how the page is used and, among other things, contains a reference count describing how many users the page has. A pointer to a `page` structure is an unambiguous way to refer to a specific physical page independent of any address space, so it is perhaps unsurprising that this structure is used with many APIs in the kernel. Should a range of memory exist that lacks corresponding `page` structures, that memory cannot be used with any API expecting a `struct page` pointer; among other things, that rules out DMA and direct I/O. 

Persistent memory looks like ordinary memory to the CPU in a number of ways. In particular, it is directly addressable at the byte level. It differs, though, in its persistence, its performance characteristics (writes, in particular, can be slow), and its size — persistent memory arrays are expected to be measured in terabytes. At a 4KB page size, billions of `page` structures would be needed to represent this kind of memory array — too many to manage efficiently. As a result, currently, persistent memory is treated like a device, rather than like memory; among other things, that means that the kernel does not need to maintain `page` structures for persistent memory. Many things can be made to work without them, but this aspect of persistent memory does bring some limitations; one of those is that it is not currently possible to perform I/O directly between persistent memory and another device. That, in turn, thwarts use cases like using persistent memory as a cache between the system and a large, slow storage array. 

#### Page-frame numbers

[One approach to the problem](<https://lwn.net/Articles/643437/>), posted by Dan Williams, is to change the relevant APIs to do away with the need for `page` structures. This patch set creates a new type called `__pfn_t`: 

```
typedef struct {
    	union {
    	    unsigned long data;
    	    struct page *page;
    	};
        __pfn_t;
```

As is suggested by the use of a union type, this structure leads a sort of double life. It can contain a `page` pointer as usual, but it can also be used to hold an integer page frame number (PFN). The two cases are distinguished by setting one of the low bits in the `data` field; the alignment requirements for `page` structures guarantee that those bits will be clear for an actual `struct page` pointer. 

A small set of helper functions has been provided to obtain the information from this structure. A call to `__pfn_t_to_pfn()` will obtain the associated PFN (regardless of which type of data the structure holds), while `__pfn_t_to_page()` will return a `struct page` pointer, but only if a page structure exists. These helpers support the main goal for the `__pfn_t` type: to allow the lower levels of the I/O stack to be converted to use PFNs as the primary way to describe memory while avoiding massive changes to the upper layers where `page` structures are used. 

With that infrastructure in place, the block layer is changed to use `__pfn_t` instead of `struct page`; in particular, the `bio_vec` structure, which describes a segment of I/O, becomes: 

```
struct bio_vec {
            __pfn_t         bv_pfn;
            unsigned short  bv_len;
            unsigned short  bv_offset;
        };
```

The ripple effects from this change end up touching nearly 80 files in the filesystem and block subtrees. At a lower level, there are changes to the scatter/gather DMA API to allow buffers to be specified using PFNs rather than `page` structures; this change has architecture-specific components to enable the mapping of buffers by PFN. 

Finally, there is the problem of enabling `kmap_atomic()` on PFN-specified pages. `kmap_atomic()` maps a page into the kernel's address space; it is only really needed on 32-bit systems where there is not room to map all of main memory into that space. On 64-bit systems it is essentially a no-op, turning a `page` structure into its associated kernel virtual address. That problem gets a little trickier when persistent memory is involved; the only code that really knows where that memory is mapped is the low-level device driver. Dan's patch set adds a function by which the driver can inform the rest of the kernel of the mapping between a range of PFNs and kernel space; `kmap_atomic()` is then adapted to use that information. 

All together, this patch set is enough to enable direct block I/O to persistent memory. Linus's [initial response](<https://lwn.net/Articles/644112/>) was on the negative side, though; he said ""I detest this approach"". Instead, he argued in favor of a solution where special `page` structures are created for ranges of persistent memory when they are needed. As the discussion went on, though, he [moderated](<https://lwn.net/Articles/644113/>) his position, saying: ""So while I (very obviously) have some doubts about this approach, it may be that the most convincing argument is just in the code."" That code has since been [reposted](<https://lwn.net/Articles/643998/>) with some changes, but the discussion is not yet finished. 

#### Back to page structures

Various alternatives have been suggested, but the most attention was probably drawn by Ingo Molnar's "[Directly mapped pmem integrated into the page cache](<https://lwn.net/Articles/644114/>)" proposal. The core of Ingo's idea is that all persistent memory would have `page` structures, but those structures would be stored in the persistent memory itself. The kernel would carve out a piece of each persistent memory array for these structures; that memory would be hidden from filesystem code. 

Despite being stored in persistent memory, the `page` structures themselves would not be persistent — a point that a number of commenters seemed to miss. Instead, they would be initialized at boot time, using a lazy technique so that this work would not overly slow the boot process as a whole. All filesystem I/O would be direct I/O; in this picture, the kernel's page cache has little involvement. The potential benefits are huge: vast amounts of memory would be available for fast I/O without many of the memory-management issues that make life difficult for developers today. 

It is an interesting vision, and it may yet bear fruit, but various developers were quick to point out that things are not quite as simple as Ingo would like them to be. Matthew Wilcox, who has done much of the work to make filesystems work properly with persistent memory, [noted](<https://lwn.net/Articles/644117/>) that there is an interesting disconnect between the lifecycle of a page-cache page and that of a block on disk. Filesystems have the ability to reassign blocks independently of any memory that might represent the content of those blocks at any given time. But in this directly mapped view of the world, filesystem blocks and pages of memory are the same thing; synchronizing changes to the two could be an interesting challenge. 

Dave Chinner [pointed out](<https://lwn.net/Articles/644118/>) that the directly mapped approach makes any sort of data transformation by the filesystem (such as compression or encryption) impossible. In Dave's view, the filesystem [needs to have a stronger role](<https://lwn.net/Articles/644120/>) in how persistent memory is managed in general. The idea of just using existing filesystems (as Ingo had suggested) to get the best performance out of persistent memory is, in his view, not sustainable. Ingo, instead, seems to feel that management of persistent memory could be mostly hidden from filesystems, just like the management of ordinary memory is. 

In any case, the proof of this idea would be in the code that implements it, and, currently, no such code exists. About the only thing that can be concluded from this discussion is that the kernel community still has not figured out the best ways of dealing with large persistent-memory arrays. Likely as not, it will take some years of experience with the actual hardware to figure that out. Approaches like Dan's might just be merged as a way to make things work for now. The best way to make use of such memory in the long term remains undetermined, though.
