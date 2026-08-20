---
title: Struct slab comes to 5.17
url: https://lwn.net/Articles/881039/
date: "January 14, 2022"
category: "Memory management-struct page"
author: "By Jonathan Corbet January 14, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
January 14, 2022

The ongoing [memory folio work](<https://lwn.net/Articles/849538/>) has caused ripples through much of the kernel and inspired a few side projects, one of which was [the removal of slab-specific fields](<https://lwn.net/Articles/871982/>) from `struct page`. That work has been pulled into the mainline for the 5.17 kernel release; it is thus a good time to catch up with the status of `struct slab` and why this work is important. 

#### `struct page` and `struct slab`

The `page` structure is at the core of the memory-management subsystem. One of these structures exists for every page of physical memory in the system; they are used to track the status of memory as it is used (and reused) during the lifetime of the system. Physical pages can adopt a number of different identities over time; they can hold user-space data, kernel data structures, DMA buffers, and so on. Regardless of how a page is used, `struct page` is the data structure that tracks its state. These structures are stored in a discontiguous array known as the system memory map. 

There are a few problems that have arisen with this arrangement. The `page` structure was significantly reorganized for 4.18, but [the definition of `struct page`](<https://elixir.bootlin.com/linux/v5.16/source/include/linux/mm_types.h#L32>) is still a complicated mess of `#ifdef`s and unions with no mechanisms to ensure that the right fields are used at any given time. The unlucky developer who needs to find more space in this structure will be hard put to understand which bits might be safe to use. Subsystems are normally designed to hide their internal data structures, but `struct page` is heavily used throughout the kernel, making any memory-management changes more complicated. One possible change — reducing the amount of memory consumed by `page` structures by getting rid of the need for a structure for every page — is just a distant dream under the current organization. 

So there are a lot of good reasons to remove information from `struct page` and hide what remains within the memory-management subsystem. One of the outcomes from [the folio discussions](<https://lwn.net/Articles/868598/>) has been a renewed desire to get a handle on `struct page`, but that is not a job for the faint of heart — or for the impatient. Many steps will be required to reach that goal. The merging of the initial folio patches for 5.16 was one such step; the advent of `struct slab` in 5.17 is another. 

There are many memory allocators inside the kernel, but two sit at the core of the memory-management subsystem and are responsible for most allocations in a running system. The page allocator, as its name suggests, deals only in units of pages; it is used when larger amounts of memory are needed. The slab allocator, instead, efficiently handles allocations of smaller objects, including those done with functions like `kmalloc()`. The slab allocator will obtain blocks of one or more pages from the page allocator, then split those blocks up and hand out the pieces as needed. There are actually three slab allocators supported by the kernel (described below), but one of them must be chosen at configuration time. 

When the slab allocator allocates pages, those pages are marked inside the associated `page` structures as being slab pages, and the interpretation of numerous fields within those structures changes. The slab-specific information does not really need to be in `struct page`, and the slab allocators shouldn't need access to the other information in that structure, but it is all mixed together anyway. 

#### Changes for 5.17

The separation of `struct slab` is a first step toward remedying this situation. For now, `struct slab` overlays the `page` structure and, thus, still uses the same memory, but the new `slab` structure hides `struct page` and constrains the slab allocators to using only the slab-specific data stored there. This work was originally done by Matthew Wilcox as part of the folio effort; it was later taken on and pushed to its conclusion by Vlastimil Babka. 

The kernel currently supports three slab allocators: SLAB (the original allocator), SLUB (a newer allocator, focused on scalability, that is normally used outside of embedded applications), and SLOB (a tiny allocator for highly memory-constrained systems). The allocator that any given kernel will use is chosen at build time using a configuration option. One of the changes Babka made to the patch set was to further narrow the definition of `struct slab` to only the fields needed for the chosen allocator. There is still only one definition with a set of `#ifdef` blocks, but it might make more sense to view the end result without them. If the SLAB allocator is selected, `struct slab` ends up looking like this: 

```
struct slab {
            unsigned long __page_flags;
            **union {
                struct list_head slab_list;
                struct rcu_head rcu_head;
            };
            struct kmem_cache *slab_cache;**
            void *freelist;    /* array of free object indexes */
            **void *s_mem;    /* first object */
            unsigned int active;**
            atomic_t __page_refcount;
        #ifdef CONFIG_MEMCG
            unsigned long memcg_data;
        #endif
        };
```

(The allocator-specific fields are shown in bold). If, instead, SLUB is configured, the structure becomes: 

```
struct slab {
            unsigned long __page_flags;
            **union {
                struct list_head slab_list;
                struct rcu_head rcu_head;
        #ifdef CONFIG_SLUB_CPU_PARTIAL
                struct {
                    struct slab *next;
                    int slabs;    /* Nr of slabs left */
                };
        #endif
            };
            struct kmem_cache *slab_cache;
            /* Double-word boundary */**
            void *freelist;        /* first free object */
            **union {
                unsigned long counters;
                struct {
                    unsigned inuse:16;
                    unsigned objects:15;
                    unsigned frozen:1;
                };
            };
            unsigned int __unused;**
            atomic_t __page_refcount;
        #ifdef CONFIG_MEMCG
            unsigned long memcg_data;
        #endif
        };
```

And for SLOB it is: 

```
struct slab {
            unsigned long __page_flags;
            **struct list_head slab_list;
            void *__unused_1;**
            void *freelist;        /* first free block */
            **long units;
            unsigned int __unused_2;**
            atomic_t __page_refcount;
        #ifdef CONFIG_MEMCG
            unsigned long memcg_data;
        #endif
        };
```

This organization helps to ensure that one slab allocator does not accidentally use fields belonging to another, yielding another increase in type safety. 

The new structure lives in `mm/slab.h`; it is _not_ under include, and thus it is not available to code outside of the memory-management subsystem. That created trouble for the x86 bootmem allocator and `zsmalloc()`, both of which were using the slab-specific fields in `struct page` even though they are not slab allocators. Those usages have been changed to other `struct page` fields; comments have also been added suggesting that this usage should be cleaned up someday. 

Meanwhile, the code within the slab allocators has been changed to use the new structure, with the conversion from `struct page` happening at the beginning of the call chains. That isolates most of the slab code from `struct page`, paving the way for future work that could separate the two structures entirely and allow `slab` structures to be allocated dynamically as needed. 

The end result is a view into the system memory map for slab allocators that begins to separate them from the lower-level memory-management details and increases type safety. Linux users, meanwhile, should see no changes other than, with luck, a reduction in the number of bugs going forward. Further in the future, there may come a time when `struct slab` can be dynamically allocated and separated entirely from the memory map. That change will be a while in coming, though; meanwhile, a cleaning up of the core memory-management types is a step in the right direction.
