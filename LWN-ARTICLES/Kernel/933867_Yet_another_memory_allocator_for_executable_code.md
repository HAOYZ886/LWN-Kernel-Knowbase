---
title: Yet another memory allocator for executable code
url: https://lwn.net/Articles/933867/
date: "June 8, 2023"
category: Memory management
author: "By Jonathan Corbet June 8, 2023"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
June 8, 2023

The kernel is an increasingly dynamic body of code, where new executable text can show up at any time. Currently, the task of allocating memory for new kernel code falls on the subsystem that first brought the ability to load code into a running kernel: the module loader. [This patch set](<https://lwn.net/ml/linux-kernel/20230601101257.530867-1-rppt@kernel.org/>) from Mike Rapoport looks to move the responsibility for these allocations to a new "JIT allocator", addressing a number of rough edges in the process. 

In order to support the ability to load modules at run time, the kernel had to gain the ability to allocate memory to hold those modules. Early on, that was just a matter of calling `vmalloc()` to obtain the requisite number of pages and enabling execute permission for the resulting pages. Over time, though, things have grown more complicated — as they so often seem to do. 

On one side, the number of subsystems that load code into a running kernel has been growing. Tracing, for example, can require adding small bits of code to the kernel. A more frequent user in current kernels is the BPF subsystem, which can see (usually) small executable segments come and go on a regular basis. The [proposed bcachefs filesystem](<https://lwn.net/ml/linux-kernel/20230509165657.1735798-1-kent.overstreet@linux.dev/>) has an even more esoteric use case; it [generates a special unpack function](<https://lwn.net/ml/linux-kernel/ZFq7JhrhyrMTNfd%2F@moria.home.lan/>) for each B-tree node, on the fly, for performance. All of these new users tend to stress the memory-management subsystem in different ways, leading to direct-map fragmentation and other performance problems. 

To that can be added the proliferation of processor architectures, some of which restrict the address ranges that can be used to hold kernel code. Various architectures have added their own overrides to the module allocator, complicating the code overall. Architecture maintainers are aggressively moving toward a strict regime where executable memory can never be writable at the same time, making it harder for an attacker to load code into the kernel. That, too, complicates the task for subsystems that need to write code into kernel memory. 

Rapoport's patch set is intended to simplify life for kernel subsystems that need to allocate memory for executable code. It replaces the existing `module_alloc()` interface with a pair of new functions: 

```
void *jit_text_alloc(size_t len);
        void jit_free(void *buf);
```

A call to `jit_text_alloc()` will return a `len`-byte range of executable memory, while `jit_free()` will return that memory to the system. The memory is initially zero-filled. On systems implementing a strict separation of executable and writable memory, it will not be possible to directly copy loadable code into this allocation; instead, one or both of these functions should be used: 

```
void jit_update_copy(void *buf, void *new_buf, size_t len);
        void jit_update_set(void *addr, int c, size_t len);
```

`jit_update_copy()` will copy executable text from `buf` into `new_buf`, which was returned from `jit_text_alloc()`, while `jit_update_set()` will set a range of that memory to a constant value. 

On some architectures, data associated with a code region must be allocated near that region; data segments for kernel modules can be subject to this requirement, for example. To ensure proper placement, memory to hold this data can be allocated with: 

```
void *jit_data_alloc(size_t len);
```

With this set of functions, kernel code can allocate and use space for new executable segments. There is still the matter of architecture-specific constraints, though. These constraints mostly take the form of rules about the placement of executable allocations in the kernel's virtual address space. Rather than have each architecture reimplement `jit_text_alloc()` to meet its special requirements, Rapoport introduced a new structure to simply describe those requirements to a central allocator: 

```
struct jit_address_space {
    	pgprot_t        pgprot;
    	unsigned long   start;
    	unsigned long   end;
    	unsigned long	fallback_start;
    	unsigned long	fallback_end;
        };
```

There are two of these structures to be provided by architecture-specific code: one describing the requirements for executable allocations, and one for data allocations. In each, the `pgprot` field describes the protections that must be applied in the page tables, while `start` and `end` delineate the address range in which the allocations should fall. Some architectures implement a second "fallback" range to be used if an allocation attempt from the primary range fails; the location of the fallback range, if any, is stored in `fallback_start` and `fallback_end`. 

These structures are then bundled into an overall structure controlling how allocations of executable memory (and associated data) are handled on any given architecture: 

```
struct jit_alloc_params {
    	struct jit_address_space	text;
    	struct jit_address_space	data;
    	enum jit_alloc_flags		flags;
    	unsigned int			alignment;
        };
```

The `flags` field allows for the expression of additional, architecture-specific quirks, while `alignment` allows the specification of the minimum alignment required for such allocations. A certain amount of digging is required to learn that `alignment` is interpreted as a power of two; alternatively, one can think of it as the number of least-significant bits that must be zero in a properly aligned address. 

With this infrastructure in place, it is possible for the kernel subsystems needing to allocate space for executable text to get the memory they need. Since this allocator is separate from the kernel's module loader, it is no longer necessary to enable loadable modules to be able to load other types of code. No real effort has been made to address the performance issues associated with the allocation of executable memory; the idea is that this sort of optimization can be added after the interface has been agreed on. 

Comments on this work have fallen into two broad categories. Rick Edgecombe [worried](<https://lwn.net/ml/linux-kernel/0f50ac52a5280d924beeb131e6e4717b6ad9fdf7.camel@intel.com/>) that this interface could expose executable code that has not yet reached its intended state. Module code, for example, can be tweaked in a number of ways after it lands in memory. It might be better, he suggested, to prepare the code area first before making it executable. 

The other [concern](<https://lwn.net/ml/linux-kernel/ZHjDU%2FmxE+cugpLj@FVFF77S0Q05N.cambridge.arm.com/>), from Mark Rutland, was that, on some architectures at least, the requirements for the placement of executable code vary depending on the type of the code. Loadable modules on arm64, for example, have tighter restrictions than kprobes do. Holding all allocations to the tightest constraints could conceivably cause an address-space shortage in the target area. He suggested creating separate allocators for each memory type, all of which might still use a common infrastructure underneath. Rapoport [answered](<https://lwn.net/ml/linux-kernel/20230605092040.GB3460@kernel.org/>) that, if it turns out to be necessary, the central infrastructure could learn to apply different rules to different allocations. It's not entirely clear, though, that the problem is serious enough to need this kind of solution. 

Overall, the patch set looks like a reasonable start toward a proper API for the allocation of executable memory in the kernel. There have been several attempts in this area over the last few years, though, and nothing has yet made everybody happy. So we'll have to wait to see what might happen this time around.
