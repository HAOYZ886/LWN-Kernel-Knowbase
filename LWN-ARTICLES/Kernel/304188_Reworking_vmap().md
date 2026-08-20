---
title: Reworking vmap()
url: https://lwn.net/Articles/304188/
date: "October 21, 2008"
category: Memory management; vmalloc
author: "By Jonathan Corbet October 21, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 21, 2008

Kernel memory is normally allocated in relatively small chunks - usually just a single page at a time. As the size of an allocation grows, satisfying that allocation with physically-contiguous pages gets progressively harder. So most of the kernel has been written with an eye toward avoiding the use of large, contiguous allocations. There are times, though, when a large memory array needs to be virtually contiguous, but not necessarily physically contiguous. One example is the allocation of space for loadable modules; any given module should live in a single, contiguous address range, but nobody cares how it's laid out in physical RAM. For cases like this, the kernel provides a set of functions like `vmalloc()` and `vmap()`. 

Functions like `vmalloc()` have long been known to be somewhat expensive to use. They have to work with a single shared (and limited) address range, and they require making changes to the kernel's page tables. Page table changes, in turn, require translation lookaside buffer (TLB) flushes, which are a costly, all-CPUs operation. So kernel developers have generally tried to avoid using these functions in performance-critical parts of the kernel. 

Nick Piggin has noticed, though, that the performance characteristics of `vmalloc()` and friends are catching up with us. The `vmalloc()` address space is kept on a linked list and protected by a global lock, which does not scale very well. But the real cost is in freeing memory regions in this space; the ensuing TLB flush must be done using an inter-processor interrupt to every CPU, each of which must then flush its own TLB. People normally do not buy more CPUs unless they have more work to run on them, so systems with more processors will, as a general rule, be performing more mapping and freeing in the `vmalloc()` range. As systems grow, there will be more global TLB flushes, each of which disrupts more processors. In other words, the amount of work grows proportional to the square of the number of processors \- meaning that everything falls down, eventually. 

To make things worse, Nick has a longstanding series of patches which, among other things, do a lot of `vmap()` calls to support larger block sizes in the filesystem layer and page cache. Merging those patches would add significantly to the amount of time the system spends managing the `vmalloc()` space, which would not be a good thing. So fixing `vmalloc()` seems like a good thing to do first. As of 2.6.28, Nick has, in fact, fixed the management of kernel virtual allocations. 

The first step is to get rid of the linked list and its corresponding global lock. Instead, a [red-black tree](<http://lwn.net/Articles/184495/>) is used to track ranges of available address space; finding a suitable region can now be done without having to traverse a long list. The tree is still protected by a global lock, which poses potential scalability problems. To avoid this issue, Nick's patch creates a separate, per-CPU list of small address ranges which can be allocated and freed in a lockless manner. New functions must be called to make use of this facility: 

```
void *vm_map_ram(struct page **pages, unsigned int count, 
                         int node, pgprot_t prot);
        void vm_unmap_ram(const void *mem, unsigned int count);
```

A call to `vm_map_ram()` will create a virtually-contiguous mapping for the given `pages`. The associated data structures will be allocated on the given NUMA `node`; the memory will have the protection specified in `prot`. With the version of the patch merged for 2.6.28, mappings of up to 64 pages can be made from the per-cpu lists. 

Note that these functions do not allocate memory, they just create a virtual mapping for a given set of pages. They are a replacement for `vmap()` and `vunmap()`, not `vmalloc()` and `vfree()`. It is probably possible to rewrite `vmalloc()` to use this mechanism, but that has not happened. So `vmalloc()` calls still require the acquisition of a global lock. 

There's another trick in this patch set which is used by all of the kernel virtual address management functions. Nick realized that it is not actually necessary to flush TLBs across the system immediately after an address range is freed. Since those addresses are being given back to the system, no code will be making use of them afterward, so it does not matter if a processor's TLB contains a stale mapping for them. All that really matters is that the TLB gets cleaned out before those addresses are used again elsewhere. So unmapped regions can be allowed to accumulate, then all flushed with a single operation. That cuts the number of TLB flushes significantly. 

How much faster do things run? Nicks patch (the merged version can be found [here](<http://git.kernel.org/git/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commit;h=db64fe02258f1507e13fe5212a989922323685ce>)) contains some benchmark results. With an artificial test aimed at demonstrating the difference, the new code runs 25 times faster. By changing the `vmap()` code in the XFS filesystem to use `vm_map_ram()` instead, some workloads were sped up by a factor of twenty. So it seems to work.
