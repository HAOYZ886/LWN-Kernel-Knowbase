---
title: The zsmalloc allocator
url: https://lwn.net/Articles/477067/
date: "January 25, 2012"
category: "Memory management-Internal API; Transcendent memory"
author: "By Jonathan Corbet January 25, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 25, 2012

The kernel cannot be said to lack for memory allocation mechanisms. At the lowest level, "memblock" handles chunks of memory for the rest of the system. The page allocator provides memory to the rest of the kernel in units of whole pages. Much of the kernel uses one of the three slab allocators to get memory blocks in arbitrary sizes, but there is also `vmalloc()` for situations where large, virtually-contiguous regions are needed. Add in various other specialized allocation functions and other allocators (like [CMA](<https://lwn.net/Articles/447405/>)) and it starts to seem like a true embarrassment of choices. So what's to be done in this situation? Add another one, of course. 

The "zsmalloc" allocator, [proposed](<https://lwn.net/Articles/474880/>) by Seth Jennings, is aimed at a specific use case. The slab allocators work by packing multiple objects into each page of memory; that works well when the objects are small, but can be problematic as they get larger. In the worst case, if a kernel subsystem routinely needs allocations that are just larger than PAGE_SIZE/2, only one object will fit within a page. Slab allocators can attempt to allocate multiple physically-contiguous pages in order to pack those large objects more efficiently, but, on memory-constrained systems, those allocations can become difficult - or impossible. So, on systems that are already tight of memory, large objects will need to be allocated one-per-page, wasting significant amounts of memory through internal fragmentation. 

The zsmalloc allocator attempts to address this problem by packing objects into a new type of compound page where the component pages are not physically contiguous. The result can be much more efficient memory usage, but with some conditions: 

  * Code using this allocator must not require physically-contiguous memory, 

  * Objects must be explicitly mapped before use, and 

  * Objects can only be accessed in atomic context. 

Code using zsmalloc must start by creating an allocation pool to work from: 

```
struct zs_pool *zs_create_pool(const char *name, gfp_t flags);
```

Where `name` is the name of the pool, and `flags` will be used to allocate memory for the pool. It is not entirely clear (to your editor, at least) why multiple pools exist; the `zs_pool` structure is relatively large, and a pool is really only efficient if the number of objects allocated from it is also large. But that's how the API is designed. 

A pool can be released with: 

```
void zs_destroy_pool(struct zs_pool *pool);
```

A warning (or several warnings) will be generated if there are objects allocated from the pool that have not been freed; those objects will become entirely inaccessible after the pool is gone. 

Allocating and freeing memory is done with: 

```
void *zs_malloc(struct zs_pool *pool, size_t size);
        void zs_free(struct zs_pool *pool, void *obj);
```

The return value from `zs_malloc()` will be a pointer value, or NULL if the object cannot be allocated. It would be a fatal mistake, though, to treat that pointer as if it were actually a pointer; it is actually a magic cookie that represents the allocated memory indirectly. It might have been better to use a non-pointer type, but, again, that is how the API is designed. Getting a pointer that can actually be used is done with: 

```
void *zs_map_object(struct zs_pool *pool, void *handle);
        void zs_unmap_object(struct zs_pool *pool, void *handle);
```

The return value from zs_map_object() will be a kernel virtual address that can be used to access the actual object. The return address is essentially a per-CPU object, so the calling code will be in atomic context until the object is freed with `zs_unmap_object()`. Note that the `handle` passed to `zs_unmap_object()` is the original cookie obtained from `zs_malloc()`, not the pointer from `zs_map_object()`. Note also that only one object can be safely mapped at a time on any given CPU. 

Internally, zsmalloc divides allocations by object size much like the slab allocators do, but with a much higher granularity - there are 254 possible allocation sizes all less than `PAGE_SIZE`. For each size, the code calculates an optimum number of pages (up to 16) that will hold an array of objects of that size with minimal loss to fragmentation. When an allocation is made, a "zspage" is created by allocating the calculated number of individual pages and tying them all together. That tying is done by overloading some fields of `struct page` in a scary way (that is not a criticism of zsmalloc: _any_ additional meanings overlaid onto the already heavily overloaded `page` structure are scary): 

  * The first page of a zspage has the `PG_private` flag set. The `private` field points to the second page (if any), while the `lru` list structure is used to make a list of zspages of the same size. 

  * Subsequent pages are linked to each other with the `lru` structure, and are linked back to the first page with the `first_page` field (which is another name for `private`, if one looks at the structure definition). 

  * The last page has the `PG_private_2` flag set. 

Within a zspage, objects are packed from the beginning, and may cross the boundary between pages. The cookie returned from `zs_malloc()` is a combination of a pointer to the `page` structure for the first physical page and the offset of the object within the zspage. Making that object accessible to the rest of the kernel at mapping time is a matter of calculating its location, then either (1) mapping it with `kmap_atomic()` if the object fits entirely within one physical page, or (2) assigning a pair of virtual addresses if the object crosses a physical page boundary. 

The primary users of zsmalloc are the zcache and zram mechanisms, both of which are currently in staging. These subsystems use the [transcendent memory](<https://lwn.net/Articles/454795/>) abstraction to store compressed copies of pages in memory. Those compressed pages can still be a substantial fraction of the (uncompressed) page size, so fragmentation issue addressed by zsmalloc can be a real problem. Given the specialized use case and the limitation imposed by zsmalloc, it is not clear that it will find users elsewhere in the kernel, but one never knows.
