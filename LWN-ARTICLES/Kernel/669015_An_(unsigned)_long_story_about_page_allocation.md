---
title: An (unsigned) long story about page allocation
url: https://lwn.net/Articles/669015/
date: "December 23, 2015"
category: "Memory management-Internal API"
author: "By Jonathan Corbet December 23, 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
December 23, 2015

The kernel project is famously willing to change any internal interface as needed for the long-term maintainability of the code. Effects on out-of-tree modules or other external code are not generally deemed to be reasons to keep an interface stable. But what happens if you want to change one of the oldest interfaces found within the kernel — one with many hundreds of call sites? It turns out that, in 2015, the appetite for interface churn may not be what it once was. 

If one looks at [`mm/memory.c`](<https://github.com/zavg/linux-0.01/blob/master/mm/memory.c>) in the Linux 0.01 release, one finds that a page of memory is allocated with: 

```
unsigned long get_free_page(void);
```

From the memory-management point of view, the system's RAM can be seen as a linear array of pages, so it can make a certain amount of sense to think of addresses as integer types — indexes into the array, essentially. Integers can also be used for arbitrary arithmetic; pointers in C can be used that way too, but one quickly gets into "undefined behavior" territory where an overly enthusiastic compiler may feel entitled to create all kinds of mayhem. So `unsigned long` was established as the return type from `get_free_page()` and, in general, as the way that one refers to an address that may appear in any place in memory. 

Fast-forward to the 4.4-rc6 release and dig through a rather larger body of code, and one finds that pages are allocated with: 

```
unsigned long __get_free_pages(gfp_t gfp_mask, unsigned int order);
        unsigned long __get_free_page(gfp_t gfp_mask);
```

The latter is a macro calling the former with an `order` of zero. Note that, more than 24 years after the 0.01 release, `unsigned long` is still used as the return type from `__get_free_pages()`. There are other variants (`alloc_pages()`, for example) that return `struct page` pointers, but much of low-level, page-oriented memory management in Linux is still done with `unsigned long` values. 

The only problem is that, often, the kernel must deal with a page of memory as _memory_ , modifying its contents. That requires a pointer. So even back in 0.01, one can find code like: 

```
p = (struct task_struct *) get_free_page();
```

The `unsigned long` return value is immediately cast into the pointer value that is actually needed. Al Viro did a survey of `__get_free_pages()` users in current kernels and [concluded](<https://lwn.net/Articles/669016/>) that ""well above 90%"" of the callers were using the return value as a pointer. That turns out to be a lot of casts, suggesting that the type of the return value for this function is not correct. So, he suggested, it might make sense to change it: 

In other words, switching to void * for return values of allocating and argument of freeing functions would reduce the amount of boilerplate quite nicely. What's more, fewer casts means better chance for typechecking to catch more bugs. 

Some of those bugs, he pointed out, he found simply by looking at the code with this kind of transformation in mind. Ten days later, he showed up with [a patch set](<https://lwn.net/Articles/668852/>) making the change and asked for a verdict from Linus. 

One might find various faults with [Linus's response](<https://lwn.net/Articles/669017/>), but a lack of clarity will not be among them. He left no doubt that there was no place in the mainline for this particular patch set. The diffstat in Al's patch (568 files changed, 1956 insertions, 2202 deletions) was clearly frightening — enough, in its own right, to rule out the change. A patch this wide-ranging would create conflicts throughout the tree and make life difficult for those backporting patches. This interface, it seems, is too old and too entrenched for this kind of flag-day change; as Linus put it: ""No way in hell do we suddenly change the semantics of an interface that has been around from basically day #1."" 

Still, as he [clarified](<https://lwn.net/Articles/669018/>) afterward, Linus isn't arguing for leaving everything exactly as it is. He accepted that most callers likely want a pointer value. But the way forward isn't to thrash up an interface like `__get_free_pages()`; instead, there are two approaches that, he said, could be taken. 

The first of these would be to create a new, pointer-oriented interface that exists in parallel with `__get_free_pages()`. Then call sites could be converted at leisure over the course of what would probably be years. 

The alternative, Linus said, is that code needing pointers could just allocate memory with `kmalloc()` instead. Once upon a time, that would not necessarily have been a good idea, since `kmalloc()` (implemented by the slab allocators) adds overhead to the page allocator and might have expanded the size of the returned memory beyond one page. Indeed, there was a period where an allocation of exactly one page would have consumed two physically contiguous pages when the slab housekeeping information was added. But those days are long in the past. In current kernels, `kmalloc()` is fast and requires little memory beyond that which is actually allocated. Indeed, Linus pointed out, `kmalloc()` may actually be faster than `__get_free_pages()` due to its use of per-CPU object caches. 

So `kmalloc()` is probably the best option for many of the call sites currently using `__get_free_pages()`. The places where it is still inappropriate will be those needing multiple-page allocations and those needing allocations that are not only page-sized but page-aligned. In those cases, Linus said, the `unsigned long` return type might not be a bad thing, since ""it's clearly not just a random pointer allocation if the bit pattern of the pointer matters."" 

After this discussion took place, Al did [a pass over](<https://lwn.net/Articles/669020/>) the `__get_free_pages()` call sites in the filesystem code and concluded that almost all of them truly would would be better off using `kmalloc()`. So the end result of this work may be a slow shift in that direction and, perhaps, the creation of a new document telling kernel developers which memory allocator they should be using in which setting.
