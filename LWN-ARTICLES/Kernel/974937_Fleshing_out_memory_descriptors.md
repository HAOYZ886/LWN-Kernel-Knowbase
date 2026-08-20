---
title: Fleshing out memory descriptors
url: https://lwn.net/Articles/974937/
date: "May 27, 2024"
category: "Memory management-Memory descriptors"
author: "By Jonathan Corbet May 27, 2024 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 27, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

One of the long-term goals of the folio conversion in the kernel's memory-management subsystem is the replacement of the `page` structure, which describes a page of physical memory, with an eight-byte "memory descriptor". This change would reduce the overhead of tracking physical memory, increase type safety, and make memory management more flexible. Thus far, though, details on what the memory-descriptor future will look like have been relatively scarce. At the [2024 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>), Matthew Wilcox led a discussion to try to fill in the picture somewhat. 

Wilcox started by saying that he has been thinking about what will happen once the folio conversion is done. The ultimate goal, he said, looks like this: 

```
struct page {
            u64 memdesc;
        };
```

The lowest four bits would be a type field saying what kind of descriptor it is; the rest would (usually) be a pointer to a type-specific structure. David Hildenbrand immediately said that what is really needed is a type hierarchy; some types have subtypes, and the kernel will surely exceed the 16 types that can be represented in those four bits at some point; 11 types have already been defined. Wilcox disagreed, noting that no new types had been added for some time and questioning whether the kernel would ever run out. I remarked that I was documenting that claim for posterity, to general laughter. 

Descriptor type zero, he said, would be a special type indicating "miscellaneous memory with no further data". It would, as it turns out, have a number of subtypes. Pages falling under this type could include those in the vmalloc range, guard pages, offline pages, and others. Bit 11 of the descriptor would be set if the page can be mapped to user space, bits 12-17 would contain the page order, and the higher bits could contain information about which node and zone contain the page. 

There was a brief discussion of how memory descriptors would be allocated; Wilcox envisioned an interface like: 

```
struct page *page = alloc_page_desc(MEMDESC_TYPE_FOLIO);
```

Jason Gunthorpe remarked that he would like to see more details on what the state transitions for memory descriptors will be. 

Wilcox moved on to discussing pages owned by the buddy allocator; they would have a descriptor that looks like: 

```
struct buddy {
            unsigned long prev;
    	unsigned long next;
        };
```

That design reduces the size of the descriptor to two 64-bit integers, which is "a step in the right direction". That information would be enough to support basic allocator operations like insertion, removal, and merging. The amount of space needed for the descriptor could be reduced by storing page-frame numbers rather than addresses. Given a willingness to limit installed memory to 2TB, the descriptor could be condensed down to eight bytes. The only problem with that idea is that systems with more than 2TB of installed memory are on the market now. 

This descriptor could be reduced further by making it contain page-frame numbers relative to the base of the zone containing the pages. At that point, each memory zone could contain 2TB of memory; with enough zones, much larger total memory sizes could be handled. Wilcox thought that this solution might come at the cost of having to add more memory zones (generally seen as undesirable), but Vlastimil Babka pointed out that large-memory machines use a NUMA architecture, so the memory is already divided into multiple zones. 

A 30-minute slot is clearly not enough to design the descriptor-based future, so it is not surprising that this discussion did not get much further. Wilcox brought it to a conclusion by saying that his goal for this year is to get rid of the `mapping` and `index` fields of `struct page`; that will require some work to fix the existing users in the kernel. Then the work of splitting the various users of `page` structures into specific descriptor types can proceed. Once approximately half of users have been converted, he will submit a patch to shrink the `page` structure; it "should all just work", he said. That will lead to the next important phase of this transition: seeing where the performance regressions are; he admitted that he does not know how that will work out. 

(Wilcox has also [put together a few wiki pages](<https://lwn.net/ml/linux-mm/ZkVRi0lk9od3SqWp@casper.infradead.org/>) on the memory-descriptor design).
