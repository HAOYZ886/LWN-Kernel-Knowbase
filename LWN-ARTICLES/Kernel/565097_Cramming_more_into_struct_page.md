---
title: Cramming more into struct page
url: https://lwn.net/Articles/565097/
date: "August 28, 2013"
category: "Memory management-struct page; Slab allocator"
author: "By Jonathan Corbet August 28, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
August 28, 2013

As a general rule, kernel developers prefer data structures that are designed for readability and maintainability. When one understands the data structures used by a piece of code, an understanding of the code itself is usually not far away. So it might come as a surprise that one of the kernel's most heavily-used data structures is also among its least comprehensible. That data structure is `struct page`, which represents a page of physical memory. A recent patch set making `struct page` even more complicated provides an excuse for a quick overview of how this structure is used. 

On most Linux systems, a page of physical memory contains 4096 bytes; that means that a typical system contains millions of pages. Management of those pages requires the maintenance of a `page` structure for each of those physical pages. That puts a lot of pressure on the size of `struct page`; expanding it by a single byte will cause the kernel's memory use to grow by (possibly many) megabytes. That creates a situation where almost any trick is justified if it can avoid making `struct page` bigger. 

Enter Joonsoo Kim, who has posted [a patch set](<https://lwn.net/Articles/564381/>) aimed at squeezing more information into `struct page` without making it any bigger. In particular, he is concerned about the space occupied by `struct slab`, which is used by the slab memory allocator (one of three allocators that can be configured into the kernel, the others being called SLUB and SLOB). A slab can be thought of as one or more contiguous pages containing an array of structures, each of which can be allocated separately; for example, the `kmalloc-64` slab holds 64-byte chunks used to satisfy `kmalloc()` calls requesting between 32 and 64 bytes of space. The associated `slab` structures are also used in great quantity; `/proc/slabinfo` on your editor's system shows over 28,000 active slabs for the ext4 inode cache alone. A reduction in that space use would be welcome; Joonsoo thinks this can be done — by folding the contents of `struct slab` into the `page` structure representing the memory containing the slab itself. 

#### What's in struct page

Joonsoo's patch is perhaps best understood by stepping through `struct page` and noting the changes that are made to accommodate the extra data. The full definition of this structure can be found in `<linux/mm_types.h>` for the curious. The first field appears simple enough: 

```
unsigned long flags;
```

This field holds flags describing the state of the page: dirty, locked, under writeback, etc. In truth, though, this field is not as simple as it seems; even the question of whether the kernel is running out of room for page flags is hard to answer. See [this article](<https://lwn.net/Articles/335768/>) for some details on how the flags field is used. 

Following `flags` is: 

```
struct address_space *mapping;
```

For pages that are in the page cache (a large portion of the pages on most systems), `mapping` points to the information needed to access the file that backs up the page. If, however, the page is an anonymous page (user-space memory backed by swap), then `mapping` will point to an `anon_vma` structure, allowing the kernel to quickly find the page tables that refer to this page; see [this article](<https://lwn.net/Articles/75198/>) for a diagram and details. To avoid confusion between the two types of page, anonymous pages will have the least-significant bit set in `mapping`; since the pointer itself is always aligned to at least a word boundary, that bit would otherwise be clear. 

This is the first place where Joonsoo's patch makes a change. The `mapping` field is not currently used for kernel-space memory, so he is able to use it as a pointer to the first free object in the slab, eliminating the need to keep it in `struct slab`. 

Next is where things start to get complicated: 

```
struct {
    	union {
    	    pgoff_t index;
    	    void *freelist;
    	    bool pfmemalloc;
    	};
    
    	union {
    	    unsigned long counters;
    	    struct {
    		union {
    		    atomic_t _mapcount;
    		    struct { /* SLUB */
    			unsigned inuse:16;
    			unsigned objects:15;
    			unsigned frozen:1;
    		    };
    		    int units;
    	        };
    	    	atomic_t _count;
    	    };
    	};
        };
```

(Note that this piece has been somewhat simplified through the removal of some `#ifdef`s and a fair number of comments). In the first union, `index` is used with page-cache pages to hold the offset into the associated file. If, instead, the page is managed by the SLUB or SLOB allocators, `freelist` points to a list of free objects. The slab allocator does not use `freelist`, but Joonsoo's patch makes slab use it the same way the other allocators do. The `pfmemalloc` member, instead, acts like a page flag; it is set on a free page if memory is tight and the page should only be used as part of an effort to free more pages. 

In the second union, both `counters` and the innermost anonymous `struct` are used by the SLUB allocator, while `units` is used by the SLOB allocator. The `_mapcount` and `_count` fields are both usage counts for the page; `_mapcount` is the number of page-table entries pointing to the page, while `_count` is a general reference count. There are a number of subtleties around the use of these fields, though, especially `_mapcount`, which helps with the management of compound pages as well. Here, Joonsoo adds another field to the second union: 

```
unsigned int active;	/* SLAB */
```

It is the count of active objects, again taken from `struct slab`. 

Next we have: 

```
union {
    	struct list_head lru;
    	struct {
    	    struct page *next;
    	    int pages;
    	    int pobjects;
    	};
    	struct list_head list;
    	struct slab *slab_page; 
        };
```

For anonymous and page-cache pages, `lru` holds the page's position in one of the least-frequently-used lists. The anonymous structure is used by SLUB, while `list` is used by SLOB. The slab allocator uses `slab_page` to refer back to the containing `slab` structure. Joonsoo's patch complicates things here in an interesting way: he overlays an `rcu_head` structure over `lru` to manage the freeing of the associated slab using read-copy-update. Arguably that structure should be added to the containing union, but the current code just uses `lru` and casts instead. This trick will also involve moving `slab_page` to somewhere else in the structure, but the current patch set does not contain that change. 

The next piece is: 

```
union {
    	unsigned long private;
    #if USE_SPLIT_PTLOCKS
    	spinlock_t ptl;
    #endif
    	struct kmem_cache *slab_cache;
    	struct page *first_page;
        };
```

The `private` field essentially belongs to whatever kernel subsystem has allocated the page; it sees a number of uses throughout the kernel. Filesystems, in particular, make heavy use of it. The `ptl` field is used if the page is used by the kernel to hold page tables; it allows the page table lock to be split into multiple locks if the number of CPUs justifies it. In most configurations, a system containing four or more processors will split the locks in this way. `slab_cache` is used as a back pointer by slab and SLUB, while `first_page` is used within compound pages to point to the first page in the set. 

After this union, one finds: 

```
#if defined(WANT_PAGE_VIRTUAL)
    	void *virtual;
        #endif /* WANT_PAGE_VIRTUAL */
```

This field, if it exists at all, contains the kernel virtual address for the page. It is not useful in many situations because that address is easily calculated when it is needed. For systems where high memory is in use (generally 32-bit systems with 1GB or more of memory), `virtual` holds the address of high-memory pages that have been temporarily mapped into the kernel with `kmap()`. Following `private` are a couple of optional fields used when various debugging options are turned on. 

With the changes described above, Joonsoo's patch moves much of the information previously kept in `struct slab` into the `page` structure. The remaining fields are eliminated in other ways, leaving `struct slab` with nothing to hold and, thus, no further reason to exist. These structures are not huge, but, given that there can be tens of thousands of them (or more) in a running system, the memory savings from their elimination can be significant. Concentrating activity on `struct page` may also have beneficial cache effects, improving performance overall. So the patches may well be worthwhile, even at the cost of complicating an already complex situation. 

And the situation is indeed complex: `struct page` is a complicated structure with a number of subtle rules regarding its use. The saving grace, perhaps, is that it is so heavily used that any kind of misunderstanding about the rules will lead quickly to serious problems. Still, trying to put more information into this structure is not a task for the faint of heart. Whether Joonsoo will succeed remains to be seen, but he clearly is not the first to eye `struct page` as a place to stash some useful memory management information.
