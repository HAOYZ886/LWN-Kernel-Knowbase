---
title: "Read-only dynamic data"
url: https://lwn.net/Articles/750215/
date: "March 27, 2018"
category: "Memory management; Security-Kernel hardening"
author: "By Jonathan Corbet March 27, 2018"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
March 27, 2018

Kernel developers go to some lengths to mark read-only data so that it can be protected by the system's memory-management unit. Memory that cannot be changed cannot be altered by an attacker to corrupt the system. But the kernel's mechanisms for managing read-only memory do not work for memory that must be initialized after the initial system bootstrap has completed. A [patch set from Igor Stoppa](<https://lwn.net/Articles/750229/>) seeks to change that situation by creating a new API just for late-initialized read-only data. 

The most straightforward way to create read-only data is, of course, the C `const` keyword. The compiler will annotate any data marked with `const`, and the linker will ensure that it is placed in memory that ends up being marked read-only. But `const` only works at build time. The [post-init read-only data mechanism](<https://lwn.net/Articles/666550/>), adapted from the grsecurity patch set, takes things a step further by marking data that can be made read-only once the system's initialization process has completed. Data structures that must be set up during boot, but which need not be modified thereafter, can be protected in this way. 

Once initialization is completed, though, the (easy) ability to create read-only data in the kernel goes away. At that point, any additional memory needed must be allocated dynamically, and such memory is, by its nature, dynamic. So, while a kernel subsystem may well allocate memory, fill it in, and never change it again, there is no mechanism in place to actually block further modifications to that memory. 

The proposed new API could be such a mechanism; it is called the "protectable memory allocator", or "pmalloc". The core concept is that a subsystem allocates a "pool" for a set of objects that will all be rendered read-only at the same time. The individual objects can be allocated from the pool and initialized; then the whole thing is set in stone. Or, in terms of code, one starts with: 

```
#include <linux/pmalloc.h>
    
        struct pmalloc_pool *pool = pmalloc_create_pool();
```

The return value (on success) is a pointer to a pool from which objects can be allocated. Thereafter, objects can be allocated from the pool with any of: 

```
void *pmalloc(struct pmalloc_pool *pool, size_t size);
        void *pzalloc(struct pmalloc_pool *pool, size_t size);
        void *pmalloc_array(struct pmalloc_pool *pool, size_t n, size_t size);
        void *pcalloc(struct pmalloc_pool *pool, size_t n, size_t size);
        char *pstrdup(struct pmalloc_pool *pool, const char *s);
```

The basic allocation function is `pmalloc()`, which allocates a chunk of memory of the given `size` from the `pool`. Variants include `pzalloc()` (which zeroes the memory before returning it), `pmalloc_array()` (to allocate an array of objects), `pcalloc()` (which perhaps should be called `pzalloc_array()`), and `pstrdup()` (which allocates memory and copies a string into it). 

When the process of allocating and initializing objects has run its course, the entire set of objects associated with the pool is made read-only with a call to: 

```
void pmalloc_protect_pool(struct pmalloc_pool *pool);
```

It's worth noting that it is still possible to allocate objects from the pool after a call to `pmalloc_protect_pool()`. The newly allocated objects will be writable until the next call. Frequent calls to `pmalloc_protect_pool()` when many objects are being allocated may result in faster protection, but it may also waste any unallocated memory within the pages that are write-protected. There is no way to "unprotect" memory once it has been made read-only; the protection of memory in pmalloc is meant to be a permanent thing. 

While there are numerous variants of `pmalloc()` to obtain protectable memory, there is no `pfree()` function. The only way to release this memory is to get rid of the entire pool with: 

```
void pmalloc_destroy_pool(struct pmalloc_pool *pool);
```

This call will return all objects allocated from `pool`, so the caller should be sure that none of them are still in use. 

Underneath this interface, pmalloc uses `vmalloc()` to obtain a range (some number of pages) of memory; that range is then managed to satisfy individual allocation requests. As a result, the pmalloc functions cannot be used in atomic context, but it is hard to imagine a situation where that would be necessary in any case. Marking the pool read-only is a matter of dipping into the internals of the memory-management layer to tweak the page-table entries for the pages holding the pool. 

The advantage of using pmalloc is that it can protect objects from unwanted modifications after they have been initialized. There is one significant limitation, though. While the protection is applied to the page-table entries in the `vmalloc()` area containing the pool, the underlying memory remains writable in the main system memory map. So an attacker who can determine where an object sits in physical memory may be able to bypass the protection applied by pmalloc entirely. Pmalloc thus places another obstacle in an attacker's path, but is not an absolute protection against modification. 

One thing that is missing from the current patch set is a set of in-kernel users of the new API. So it is not entirely clear where Stoppa intends this mechanism to be used. That omission will probably have to be rectified at some point; the kernel community is reluctant to merge new APIs without in-tree users to show how those APIs work in real-world use. Once that has been filled in, the community should have some time to debate the merits of this interface before the 4.18 development cycle begins.
