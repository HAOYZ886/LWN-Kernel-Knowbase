---
title: Moving x86 to LMB
url: https://lwn.net/Articles/387083/
date: "May 11, 2010"
category: "Logical memory block LMB; Memory management-During early boot"
author: "By Jonathan Corbet May 11, 2010"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
May 11, 2010

The early days of the 2.6.34 development cycle were made more difficult for some testers by difficulties in the [NO_BOOTMEM patches](<http://lwn.net/Articles/382559/>) which came in during the merge window. The kinks in that code were eventually ironed out, but things might just get interesting again in 2.6.35 - Yinghai Lu is back with [another set of patches](<http://lwn.net/Articles/386864/>) which continues the process of completely reworking how early memory allocation is done on the x86 architecture. The potential for trouble with this kind of work is always there, but the end result does indeed seem worth aiming for. 

Some review: in a running kernel, memory management is handled by the buddy allocator (at the page level), with the slab allocator on top. These allocators are complex pieces of code which cannot run in the absence of a mostly functional kernel, so they cannot be used in the early stages of the bootstrap process. What is used, instead, is an architecture-specific chain of simple allocators. For x86, things start with a `brk()`-like mechanism which yields to the "e820" early reservation code, which, in turn, gives way to the bootmem allocator. Once the bootstrap has gotten far enough, the slab allocator can take over from the bootmem code. Yinghai's 2.6.34 changes were meant to short out the bootmem stage, allowing the system to use the early reservation code until slab can run. 

During the review process for that code, some reviewers asked why x86 did not use the "logical memory block" (LMB) allocator instead of its own early reservation code. LMB is currently used by the Microblaze, PowerPC, SuperH, and SPARC architectures, so it has the look of a generic solution. There are obvious advantages to using generic code over architecture-specific variants; there are more eyes to look at the code and the overall maintenance cost is reduced. So the idea of moving to LMB made obvious sense. 

LMB is, as might be expected, a truly simplistic memory manager. Low-level architecture code gives it blocks of memory to manage as it discovers them with: 

```
long lmb_add(u64 base, u64 size);
```

The LMB allocator will duly store that region into a fixed-length array of known memory blocks, coalescing it with existing blocks if need be. Memory may then be allocated with: 

```
u64 lmb_alloc(u64 size, u64 align);
```

Allocated blocks are tracked in a second array which looks just like the first; an allocation is satisfied by iterating through the available blocks, trying to find a sufficiently large chunk which is not already reserved by somebody else. There are other functions for reserving specific regions of memory, allocating memory on specific NUMA nodes, etc. But, at its core, LMB is a simple allocator which is meant to do a good-enough job until something more sophisticated can take over. 

Yinghai's patch set makes a number of changes to the LMB code itself, starting with a move from the `lib` directory over to `mm` with the rest of the memory-management code. Some new functions are added to match the different semantics supported by the early reservation code, which works in a two-step, "find a memory block, then reserve it" mode. There is also a new function to transfer LMB reservations into the bootmem allocator for configurations where bootmem is still in use. The 22-part series culminates with a switch to LMB calls for early allocations and the removal of the now-unused early reservation code. 

There has been surprisingly little discussion for a patch series which makes such fundamental changes. It seems that most kernel developers pay relatively little attention to what happens at the architecture-specific levels. One exception is Ben Herrenschmidt, who keeps an eye on LMB from the PowerPC perspective. Ben disagrees with a number of the LMB-level changes, feeling that they complicate the API and potentially introduce problems. Instead, it looks like Ben would like to fix up the LMB code himself, letting Yinghai work on the x86-specific side of things. 

To that end, Ben has posted [a patch series of his own](<http://lwn.net/Articles/386991/>), saying: 

My aim is still to replace the bottom part of Yinghai's patch series rather than build on top of it, and from there, add whatever he needs to successfully port x86 over and turn NO_BOOTMEM into something half decent without adding a ton of unneeded crap to the core lmb. 

Some of the changes simply clean up the LMB code, adding, for example, a `for_each_lmb()` macro for iterating through the array of memory blocks. The fixed-length arrays are made variable, `phys_addr_t` is used to represent physical addresses, and the code is substantially reorganized. There is much that Ben still plans to do, including, happily, the addition of actual documentation to the API, but even without all that, it's a significant cleanup for the LMB code. 

As with Yinghai's patches there has been little in the way of discussion. It may be that these changes will remain below the radar while the two patch sets are integrated and - maybe - merged for 2.6.35. With luck, they'll remain below the radar thereafter as well, with few people even noticing the difference.
