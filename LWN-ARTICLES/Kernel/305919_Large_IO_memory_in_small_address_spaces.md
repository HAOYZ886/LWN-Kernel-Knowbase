---
title: Large I/O memory in small address spaces
url: https://lwn.net/Articles/305919/
date: "November 4, 2008"
category: "Device drivers-Video; IO memory; Memory management-Video memory"
author: "By Jonathan Corbet November 4, 2008"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 4, 2008

In the good old days, video graphics drivers ran in user space and the kernel had little to do with video memory. More recently, graphics developers have decisively voted for change and, in the process, moved video memory management into the kernel. So now the kernel must often manipulate video memory directly. And that, as it turns out, is harder than one might expect - at least, on 32-bit machines if the user actually cares about reasonable performance. 

The problem is that 32-bit machines have a mere 4GB of virtual address space. Linux (usually) splits that space in two; the bottom 3GB are given to user space, while the kernel itself occupies the top 1GB. Splitting the space in this way yields an important advantage: there is no need to adjust the memory management configuration on transitions between kernel and user space, which speeds things up considerably. The down side is that the kernel has to fit in the remaining gigabyte of memory. That would not seem like much of a problem, even with contemporary kernels, but remember one thing: the kernel needs to map physical memory into its address space before it can do anything with it. So the amount of virtual address space given to the kernel limits the amount of physical memory it can manipulate directly. 

One other thing that must fit into the kernel's address space is the `vmalloc()` area - a range of addresses which can be assigned on the fly to create needed mappings in the kernel. When a virtually-contiguous range of memory is allocated with `vmalloc()`, it is mapped in this range. Another user of this address space is `ioremap()`, which makes a range of I/O memory available to the kernel. 

Device drivers typically need access to I/O memory, so they use `ioremap()` to map it into the kernel's address space. Graphics adapters are a little different, though, in that they have _large_ I/O memory regions: the entirety of video memory. Contemporary graphics adapters can carry a lot of video memory, to the point that mapping it with `ioremap()` would require far too much address space, if, indeed, it fits in there at all. So a straight `ioremap()` is not feasible; life was much easier in the old days when this I/O memory was mapped into user space instead. 

The Intel i915 developers, who are the farthest ahead when it comes to kernel-based GPU memory management, ran into this problem first. Their initial solution was to map individual pages as needed with `ioremap()` (or, strictly, `ioremap_wc()`, which turns on write combining - see [this article](<http://lwn.net/Articles/282250/>) for more details), and unmapping them afterward. This solution works, but it's slow. Among other things, an `ioremap()` operation requires a cross-processor interrupt to be sure that all CPUs know about the address space change. It is a function which was designed to be called infrequently, outside of performance-critical code. Making `ioremap()` calls a part of most graphical operations is not the way to obtain a satisfactory first-person shooter experience. 

The real [solution](<http://lwn.net/Articles/305734/>) comes in the form of a new mapping API developed by Keith Packard (and subsequently tweaked by Ingo Molnar). It draws heavily on the fact that Linux has had to solve this kind of problem before. Remember that the kernel (on 32-bit systems) only has 1GB of address space to work with; that is the maximum amount of physical memory it can ever have directly mapped at any given time. Any physical memory above that amount is called "high memory"; it is normally not mapped into the kernel's address space. Access to that memory requires an explicit mapping - using `kmap()` or `kmap_atomic()` \- first. High memory is thus trickier to use, but this trick has enabled 32-bit systems to support far more memory than was once thought possible. 

The new mapping API draws more than inspiration from the treatment of high memory - it uses much of the same mechanism as well. A driver which needs to map a large I/O area sets up the mapping with a call to: 

```
struct io_mapping *io_mapping_create_wc(unsigned long base,
    					    unsigned long size);
```

This function returns the `struct io_mapping` pointer, but it does not actually map any of the I/O memory into the kernel's address space. That must be done a page at a time with a call to one of: 

```
void *io_mapping_map_atomic_wc(struct io_mapping *mapping,
    				   unsigned long offset);
        void *io_mapping_map_wc(struct io_mapping *mapping, unsigned long offset);
```

Either function will return a kernel-space pointer which is mapped to the page at the given `offset`. The atomic form is essentially a `kmap_atomic()` call - it uses the `KM_USER0` slot, which is a good thing for developers to know about. It is, by far, the faster of the two, but it requires that the mapping be held by atomic code, and only one page at a time can be mapped in this way. Code which might sleep must use `io_mapping_map_wc()`, which currently falls back to the old `ioremap_wc()` implementation. 

Mapped pages should be unmapped when no longer needed, of course: 

```
void io_mapping_unmap_atomic(void *vaddr);
        void io_mapping_unmap(void *vaddr);
```

There are some interesting aspects to this implementation. One is that `struct io_mapping` is never actually defined anywhere. The code need not remember anything except the base address, so the return value from `io_mapping_create_wc()` is just the `base` pointer which was passed in. The other is that all of this structure is really only needed on 32-bit systems; a 64-bit processor has no trouble finding enough address space to map video memory. So, on 64-bit systems, `io_mapping_create_wc()` just maps the entire region with `ioremap_wc()`; the individual page operations are no-ops. 

Keith [reports](<https://lwn.net/Articles/305924/>) that, with this change, Quake 3 (used for testing purposes only, of course) runs 18 times faster. The far more serious Dave Airlie [tested with glxgears](<https://lwn.net/Articles/305925/>) and got an increase from 85 frames/second to 380. This is a big enough improvement that they would like to see this code go into 2.6.28, which will contain the GEM memory manager code. Linus [responds](<https://lwn.net/Articles/305926/>): 

I'm inclined to agree. Not that I think 380fps sounds very impressive (I get 850+ fps with _software_ rendering, for chissake), but because 85 fps is a joke, and clearly without this setup there's not even any point to try to do any other optimizations. 

As a result, this code has been merged into the mainline and will appear in 2.6.28-rc4.
