---
title: Alignment guarantees for kmalloc()
url: https://lwn.net/Articles/787740/
date: "May 8, 2019"
category: "Memory management-Internal API"
author: "By Jonathan Corbet May 8, 2019 LSFMM"
---

By **Jonathan Corbet**  
May 8, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

`kmalloc()` is one of the kernel's fundamental memory-allocation primitives for relatively small objects. Most of the time, developers don't worry about the alignment of memory returned from `kmalloc()`, and things generally just work. But, Vlastimil Babka said during a plenary session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, every now and then `kmalloc()` will do something surprising. He proposed tightening the guarantees around object alignment in the hope of generating fewer surprises in the future. 

In particular, Babka wanted to discuss when `kmalloc()` should return objects with their "natural alignment". What that term means was not actually defined until near the end of the session; presumably nobody will object to a slight bit of reordering at this point. Natural alignment for an object means that its beginning address is a multiple of each of up to three different quantities: `ARCH_KMALLOC_MINALIGN` (eight bytes, by [![\[Vlastimil Babka\]](https://static.lwn.net/images/conf/2019/lsfmm/VlastimilBabka-sm.jpg)](<https://lwn.net/Articles/787746/>) default), the cache-line size (for larger objects), and the size of the object itself when that size is a power of two. The actual required alignment will generally be the least common multiple of the three. 

Most of the time, Babka said, `kmalloc()` already returns naturally aligned objects for a power-of-two allocation size; that result falls out of how the slab pages are laid out. But there are exceptions: when SLUB debugging is enabled or when the SLOB allocator is used. Few people worry much about SLOB, but the SLUB debugging exception can lead to problems for unsuspecting developers. 

That is because code can come to depend on receiving objects with natural alignment without the developer realizing; then, one day, they turn on SLUB debugging and things break. Babka has found a number of Stack Overflow questions about the alignment guarantees for `kmalloc()`, often with incorrect answers. XFS, in particular, is subject to breaking due to the block-size alignment requirement it has for some objects. There are some workarounds in place, but even then, XFS is depending on implementation details rather than explicit guarantees. These workarounds also increase memory use unnecessarily. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

James Bottomley asked why XFS has this alignment requirement; Christoph Hellwig responded that XFS is "just the messenger" with regard to this problem. Another source of trouble, he said, is any of a number of devices with strange alignment requirements for I/O buffers. Matthew Wilcox said, somewhat sarcastically, that it would be great if the block layer could do bounce buffering for such devices; Hellwig responded seriously that developers are trying to reduce bounce buffering, not increase it. Besides, there are other places using memory from `kmalloc()` for I/O; any of them could show the problem. 

One possible solution, for code where there are known alignment requirements, would be to use `kmem_cache_alloc()` to create a cache with an explicit alignment size. But that requires creating caches ahead of time for each possible allocation size. Dave Hansen said that, given that the default is to provide good behavior, perhaps caches created with explicit alignment could be coalesced back into the default caches, thus reducing the overhead Babka said that would be messy to implement; the group discussed the feasibility of this idea for a while without coming to any real conclusions. 

Another possibility, Babka said, would be to create a new `kmalloc_aligned()` function that would take an explicit alignment parameter. That would be useful in situations where the required alignment is larger than the natural size. Developers would have to know about it (and about their alignment requirements) to use it, though, suggesting that it might not be used in places where it is needed. 

So, Babka said, `kmalloc()` should simply be defined to return naturally aligned objects when the allocation size is a power of two. No changes would be required to implement that guarantee with SLAB or with SLUB without debugging enabled. In the SLUB-with-debugging case, there would be a cost in the form of more wasted memory. Implementing the alignment guarantee with SLOB would end up fragmenting the heap more than it already is. 

Wilcox said that there wasn't really a need for SLUB red-zone debugging (which traps accesses outside of the allocated object) now that KASAN works; perhaps it could just be deleted? It seems, though, that other developers still find this feature useful. KASAN has a high overhead and must be built into the kernel; red-zones can be enabled at run time. Ted Ts'o said that, while he is willing to turn on red zones, he finds KASAN too painful to enable most of the time. Hellwig said he turns on red zones whenever somebody comes up with "a crazy new data structure". 

While providing an alignment guarantee would make life easier for `kmalloc()` users, Babka said, there are some costs as well. Kernels with SLUB debugging enabled would be less efficient and SLOB would be less efficient overall. The guarantee could also restrict possible improvements to the slab allocators in the future. Christoph Lameter said that it could make storing metadata with objects harder. 

Bottomley suggested a variant: natural alignment for objects up to 512 bytes, but a maximum of 512-byte alignment for larger objects up to the size of a page. Anything at least as large as one page would have page alignment. Babka said that this kind of special rule would only serve to complicate the implementation. Ts'o said that it is not worth trying too hard to avoid every potential cost of an alignment guarantee; spending a little more memory for more predictable behavior is worth it. 

Hugh Dickins worried that SLOB users, who tend to be running on memory-constrained devices, might suffer from this change. Their memory overhead will get higher, and the memory-management developers will not hear about it for a long time. Babka replied, though, that the situation is not that bad; there will be more holes in the heap, but they will end up being filled by smaller objects. Wilcox noted, though, that SLOB uses the four bytes ahead of each object to store its size; doing that while aligning user data will be "obnoxious". Babka closed the session by saying that he would look more closely at the actual overhead with SLOB, saying that it would be unfortunate if the little-used SLOB allocator were to block this project.
