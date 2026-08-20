---
title: The proper time to split struct page
url: https://lwn.net/Articles/937839/
date: "July 14, 2023"
category: "Memory management-Memory descriptors; Releases-6.6"
author: "By Jonathan Corbet July 14, 2023"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 14, 2023

The `page` structure sits at the core of the kernel's memory-management subsystem; one such structure exists for every page of installed RAM. This structure is increasingly seen as a problem, though, and phasing it out is one of the many side projects associated with the [folio conversion](<https://lwn.net/Articles/849538/>). One step in that direction is currently meeting some pushback from memory-management developers, though, who think that some of these changes are coming too soon. 

The purpose of `struct page` is to allow the kernel to keep track of the status of each page — how it is being used, its position in a least-recently-used list, how many references to it exist, and more. The information needed varies considerably depending on how a given page is being used; a page of user-space anonymous memory is managed differently from, say, memory used for a kernel-space DMA buffer. Since `page` structures must be kept as small as possible — there are millions of them in a modern system, so every byte hurts — data must be stored as efficiently as possible. As a result, `struct page` is [declared](<https://elixir.bootlin.com/linux/v6.5-rc1/source/include/linux/mm_types.h#L74>) as a maze of nested unions, allowing the data fields for each usage type to be overlaid. 

This all leads to a structure that is too big; about 1.6% of the memory in a system is used just to track that memory at the lowest level. Many uses do not require all of the space that `struct page` provides, but the size of the structure cannot vary and the extra memory is wasted. At the same time, `struct page` is too small, requiring [constant efforts to shoehorn](<https://lwn.net/Articles/787338/>) another bit into it. The structure itself is nearly incomprehensible to human minds, even after efforts have been made to clean up its definition. Which fields are available for a given memory type is not always clear. This structure also exposes a lot of internal memory-management details that would be better hidden within the memory-management subsystem, making many changes harder than they should be. 

One of the many goals of the current churn in that subsystem is to get rid of `struct page` in its current form. The system's memory map, which is currently an array of these structures, would be reduced to an array of pointers, each of which would point to a descriptor of a type suited to the current use of the page it represents. Those descriptors would be dynamically allocated and sized appropriately for the information they need to contain. 

This is not a simple change to make; since this structure has been exposed to the entire kernel, there is code all over the place that deals with it directly. This includes a lot of device drivers. Changing all of that code will not be done in a day — or in a year, for that matter. 

Thus, smaller steps need to be taken on the way toward this goal. One of those steps is for code to stop dealing with `struct page` directly and, instead, work with a usage-specific structure type. The 5.17 kernel saw [the introduction of `struct slab`](<https://lwn.net/Articles/881039/>), which describes a page of memory managed by the slab allocator. This structure overlays `struct page` exactly and is carefully designed to avoid stepping on the fields of that structure that have other uses. This change doesn't change the fact that the information lives in the same `page` structure as before, but it makes the slab-specific parts explicit and hides the rest of `struct page` from the slab allocator. 

The next step may be the [`struct ptdesc` proposal](<https://lwn.net/ml/linux-mm/20230627031431.29653-1-vishal.moola@gmail.com/>) from Vishal Moola. This structure describes the form that `struct page` takes when the memory it describes holds a page table: 

```
struct ptdesc {
        	unsigned long __page_flags;
        
        	union {
        	    struct rcu_head pt_rcu_head;
        	    struct list_head pt_list;
        	    struct {
        		unsigned long _pt_pad_1;
        		pgtable_t pmd_huge_pte;
        	    };
        	};
        	unsigned long _pt_s390_gaddr;
        
        	union {
        	    struct mm_struct *pt_mm;
        	    atomic_t pt_frag_refcount;
        	};
        
        	union {
        	    unsigned long _pt_pad_2;
        #if ALLOC_SPLIT_PTLOCKS
        	    spinlock_t *ptl;
        #else
        	    spinlock_t ptl;
        #endif
        	};
        	unsigned int __page_type;
        	atomic_t _refcount;
        #ifdef CONFIG_MEMCG
        	unsigned long pt_memcg_data;
        #endif
        };
```

As can be seen, even after this use case has been separated from the rest of of `struct page`, a number of unions remain. Many of them represent architecture-specific usages; `pt_mm` is used on x86 systems, for example, while `pt_frag_refcount` is needed on PowerPC and s390. But this structure is still much simpler, and it makes the page-table-specific usage clearer and more explicit. 

This work is in its sixth revision, and most of the concerns that have been raised about it would appear to have been addressed. This time around, though, Hugh Dickins [complained](<https://lwn.net/ml/linux-mm/e8992eee-4140-427e-bacb-9449f346318@google.com/>), saying: ""I don't see the point of this patchset: to me it is just obfuscation of the present-day tight relationship between page table and struct page."" He went on to say that, ""in a kindly mood"", he would describe the work as being ahead of its time, but would be willing to live with it if need be. David Hildenbrand [added](<https://lwn.net/ml/linux-mm/26282cb8-b6b0-f3a0-e82d-b4fec45c5f72@redhat.com/>) that he is ""not a friend of these 'overlays'"", adding that they only make sense once the descriptors can be dynamically allocated. Both developers seem to see this work as churning the memory-management code without providing any immediate benefit. 

Matthew Wilcox [answered](<https://lwn.net/ml/linux-mm/ZJyBHdcjuaykIRG9@casper.infradead.org/>) that one reason to do this work now is to better document how each usage type manages the `page` structure: 

> By creating specific types for each user of struct page, we can see what's actually going on. Before the ptdesc conversion started, I could not have told you which bits in struct page were used by the s390 code. I knew they were playing some fun games with the refcount (it's even documented in the s390 code!) but I didn't know they were using ... whatever it is; page->private to point to the kvm private data? 

There are, he said, assertions being added to ensure that the usage-specific structures continue to line up properly with `struct page` on each architecture; these can be seen in the form of the `TABLE_MATCH()` macros toward the end of [this patch](<https://lwn.net/ml/linux-mm/20230627031431.29653-4-vishal.moola@gmail.com/>) from Moola's series. 

While there seems to be a consensus among the memory-management developers regarding the replacement of `struct page` with dynamically allocated, usage-specific descriptors, there apparently has not been a conversation about the order in which those changes should take place. It might be possible to do the dynamic allocation first, but that, too, would be a lot of code churn without a huge immediate benefit. Both transformations are needed to get to where the developers are trying to go. This work has started by adding the new structure types first; chances are it will continue that way for the duration (with, perhaps, [zsmalloc descriptors](<https://lwn.net/ml/linux-kernel/20230713042037.980211-1-42.hyeyoo@gmail.com/>) being the next step).
