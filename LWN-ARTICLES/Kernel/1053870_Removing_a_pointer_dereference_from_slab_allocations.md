---
title: Removing a pointer dereference from slab allocations
url: https://lwn.net/Articles/1053870/
date: "January 15, 2026"
category: "Memory management-Slab allocators; Run-time constants"
author: "By Jonathan Corbet January 15, 2026"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 15, 2026

Al Viro does not often stray outside of the core virtual filesystem area; when he does, it is usually worthy of note. Recently, he wandered into memory management with [this patch series](<https://lwn.net/ml/all/20260110040217.1927971-1-viro@zeniv.linux.org.uk>) to the slab allocator and some of its users. Kernel developers will often put considerable effort into small optimizations, but it is still interesting to look at just how much effort has gone toward the purpose of avoiding a single pointer dereference in some memory-allocation hot paths. 

#### The slab cache

The kernel's [slab allocator](<https://docs.kernel.org/core-api/mm-api.html#the-slab-cache>) exists to provide quick allocations of fixed-sized objects. For example, the kernel uses large numbers of [`dentry`](<https://elixir.bootlin.com/linux/v6.18.4/source/include/linux/dcache.h#L92>) structures to cache information about file names; on the system where this is being written, there are currently over 800,000 active `dentry` structures, as reported by `/proc/slabinfo`. Requests to allocate and free these structures are frequent, so their performance matters. 

The slab allocator provides a function, [`kmem_cache_create()`](<https://docs.kernel.org/core-api/mm-api.html#c.kmem_cache_create>), that returns a pointer to a newly allocated and initialized `kmem_cache` structure. This pointer, in turn, can be used (by calling [`kmem_cache_alloc()`](<https://docs.kernel.org/core-api/mm-api.html#c.kmem_cache_alloc>)), to allocate a new object of the size that this particular cache was configured for. The virtual filesystem layer, for example, can use a slab cache to allocate `dentry` structures. The slab allocator will maintain a cache of available structures, handing them out on request; it will also make an effort to lay them out optimally in pages of memory obtained from the page allocator. Even simple operations in the kernel may involve allocating and freeing a number of objects, so considerable effort has gone into optimizing the slab allocator over time. 

While slab caches can be created and destroyed dynamically, there are a number of them that exist for the life of the system. The cache for `dentry` structures, for example, is created during the system bootstrap process, and the `struct kmem_cache` pointer for this cache is stored (as `dentry_cache`) in memory that is made read-only once the bootstrap is complete. Code that needs to allocate or free a `dentry` structure will, once compiled, contain the address of `dentry_cache`, which can be used to fetch the pointer to the `kmem_cache` structure that must be passed to the slab allocator. Most of the time, this extra dereference will be a small cost relative to the cost of allocating a new object but, according to Viro, it does have a measurable effect for heavily used caches. 

Optimizing out that dereference thus has some appeal, and it should be possible. The value of the `dentry_cache` pointer is constant; once it has been set, it will not change for the life of the system. All that is needed is to replace the address of `dentry_cache`, in every place in the kernel binary where it appears, with the address of the `kmem_cache` structure that is stored there. 

#### Run-time constants

The above description of how the dentry cache slab is accessed is not, as it turns out, fully accurate for current kernels. If one looks in [`fs/dcache.c`](<https://elixir.bootlin.com/linux/v6.19-rc4/source/fs/dcache.c#L89>) in a 6.19-rc kernel, one will see that the slab pointer is declared as: 

```
static struct kmem_cache *__dentry_cache __ro_after_init;
        #define dentry_cache runtime_const_ptr(__dentry_cache)
```

The pointer to the slab cache used for `dentry` structures is actually stored in a variable called `__dentry_cache`; the unadorned `dentry_cache` name is created by the `#define` in the second line. This declaration sequence demonstrates the "run-time constant" mechanism that was added to the 6.11 kernel by Linus Torvalds. He never quite got around to documenting this new feature — surely it must be near the top of his "to-do" list at this point — so one has to reverse engineer it. In short, run-time constants do what was described above: they patch an address directly into the code, at run time, so that a dereference operation can be avoided. 

To set up a pointer as a run-time constant, the first step is to declare it using the `runtime_const_ptr()` macro as seen above. That macro returns a value that, using `#define`, is bound to the name that the rest of the code uses for the pointer (`dentry_cache`, without underscores, in this case). There are other macros used to set the value of a run-time constant; for the `dentry` slab, the constant is set, in [`dcache_init()`](<https://elixir.bootlin.com/linux/v6.19-rc4/source/fs/dcache.c#L3261>), using `runtime_const_init()`: 

```
__dentry_cache = KMEM_CACHE_USERCOPY(dentry,
    	    SLAB_RECLAIM_ACCOUNT|SLAB_PANIC|SLAB_ACCOUNT,
    	    d_shortname.string);
        runtime_const_init(ptr, __dentry_cache);
```

The `kmem_cache` is allocated with [`KMEM_CACHE_USERCOPY()`](<https://elixir.bootlin.com/linux/v6.18.4/source/include/linux/slab.h#L484>), which is a macro wrapping `kmem_cache_create()`; the resulting pointer is then used to set the value of the run-time constant. This initialization will cause any instruction in the kernel code that references `dentry_cache` to contain the `kmem_cache` pointer instead. So the extra dereference is eliminated; this was the motivation for the addition of the run-time constant machinery in the first place. 

So it might appear that the problem is already solved, but there are a couple of ways in which this solution falls short. The first is that not all architectures support run-time constants, though it seems that the most important architectures do. But this mechanism only works during the system bootstrap process; once the system is fully booted, it is no longer possible to modify the kernel text to reflect the actual value of a run-time constant. That, in turn, means that run-time constants cannot be used in loadable modules. 

#### Static `kmem_cache` structures

Rather than try to fix run-time constants to address those problems, Viro decided to focus on the slab problem specifically. It is easy enough to have kernel code contain a pointer to the `kmem_cache` structure it needs to use, without the need for run-time code patching, if that structure is allocated statically on the caller's side. The address of that structure becomes a compile-time constant. Even loadable modules would be able to use such a feature, at least for slabs allocated and managed within the module itself. 

One small obstacle that needs to be overcome is that the definition of `struct kmem_cache` is hidden from code outside of the slab allocator itself, and for good reasons. That will make it hard to declare those structures elsewhere in the kernel. The key to the solution is the realization that this code really only needs to allocate some memory that is large enough to hold a `kmem_cache` structure. So Viro's patch set introduces a new type, `struct kmem_cache_opaque`, that is defined in such a way that it is the same size as `struct kmem_cache`, but which does not reveal any of the details of that structure. There is a new macro, `to_kmem_cache()`, that will cast a pointer to the opaque form of the structure to the regular type expected by the slab subsystem. 

With these changes, the declaration of `dentry_cache` becomes: 

```
static struct kmem_cache_opaque __dentry_cache;
        #define dentry_cache to_kmem_cache(&__dentry_cache)
```

A few other changes are needed to convert a subsystem over to a static `kmem_cache` structure. The usual call to `kmem_cache_create()` becomes, instead, a call to `kmem_cache_setup()` with the same parameters. (In the dentry cache, the more specialized `KMEM_CACHE_USERCOPY()` macro becomes `KMEM_CACHE_SETUP_USERCOPY()`). Otherwise, code that allocates and frees objects works without change. 

Making this feature work in modules required a bit more plumbing to ensure that the cleanup of statically allocated slab caches is completed before the module that created those caches is removed. Within the slab allocator itself, the main change was to make note of preallocated `kmem_slab` structures so that the slab code does not try to allocate or free them itself. Statically allocated slab caches also cannot be merged with any others. 

The patch series converts a fair number of caches in the core kernel and filesystem subsystems to the static variant. There are no benchmark results showing how much of a performance improvement ensues. Torvalds [was happy](<https://lwn.net/ml/all/CAHk-=wiibHkNcsvsVpQLCMNJOh-dxEXNqXUxfQ63CTqX5w04Pg@mail.gmail.com>) with the patch set, calling it ""much better than runtime_const for these things"". Thus far, there have not been many comments from the memory-management developers. Assuming they have no complaints, the path for this work into the mainline looks relatively smooth.
