---
title: Hardened usercopy whitelisting
url: https://lwn.net/Articles/727322/
date: "July 7, 2017"
category: "Security-Kernel hardening; copy  user"
author: "By Jonathan Corbet July 7, 2017"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 7, 2017

There are many ways to attempt to subvert an operating-system kernel. One particularly effective way, if it can be arranged, is to attack the operations that copy data between user-space and kernel-space memory. If the kernel can be fooled into copying too much data back to user space, the result can be an information-disclosure vulnerability. Errors in the other direction can be even worse, overwriting kernel memory with attacker-controlled data. The kernel has gained some defenses against this sort of attack in recent development cycles, but there is more work yet to be merged. 

Much of the heap memory used within the kernel is obtained from the slab allocator. The [hardened usercopy patch set](<https://lwn.net/Articles/695991/>), merged for the 4.8 kernel, attempts to limit the impact of erroneous copy operations by ensuring that no single operation can cross the boundary between one slab-allocated object and the next. But the kernel gets a lot of large memory objects from the slab allocator, and it is often not necessary to copy the entire object between the kernel and user space. In cases where only part of an object needs to be copied, it would be useful to prevent a rogue copy operation from copying to or from parts of the structure that do not need to be exposed in this way. 

For example, the large [`mm_struct` structure](<http://elixir.free-electrons.com/linux/latest/source/include/linux/mm_types.h#L359>) describes a process's virtual address space; it contains quite a bit of security-sensitive information. One field in this structure, `saved_auxv` is copied to and from user space. The `prctl()` functions that manipulate this field do not copy directly to or from the structure, but there is [some obscure code](<http://elixir.free-electrons.com/linux/latest/source/fs/binfmt_elf.c#L337>) in the ELF binary-format code that does pass that field directly to `copy_to_user()`. It would be nice if that copy operation could be restricted to that one field without the risk of exposing the rest of the structure. 

Enabling protection at that level is the purpose of the [hardened usercopy whitelisting](<https://lwn.net/Articles/725935/>) patch set. Experience says we need to get the provenance of such patches right, so: this code originally comes from the grsecurity/PaX patch sets. David Windsor ported and reworked the code for mainline, and Kees Cook posted the set for review. 

In short, this patch set extends the hardened usercopy mechanism by allowing the specification of a "usercopy region" within a slab-allocated object. Only data within that region can be copied to and from user space with functions like `copy_to_user()` or `copy_from_user()`. It is worth noting that no checking is applied to primitives like `put_user()`; the size of those operations is fixed and should not be subject to run-time attack. 

Normally, a slab cache is allocated with `kmem_cache_create()`. This patch set adds a new function: 

```
struct kmem_cache *kmem_cache_create_usercopy(const char *name,
    			    size_t obj_size, size_t align, unsigned long flags,
    			    size_t useroffset, size_t usersize,
    			    void (*ctor)(void *));
```

The `useroffset` and `usersize` parameters are new in this version of the function; they describe the region of objects allocated from this cache that can be copied between kernel and user space. If `usersize` is zero, no copying is allowed at all. Slabs created with `kmem_cache_create()` and objects obtained with functions like `kmalloc()` are fully whitelisted. 

Whenever an object obtained from a slab allocator is passed to one of the user-space copy functions, the area to be copied will be checked to ensure that it lies entirely within the whitelisted window. If that test fails, a kernel oops will result. 

One implication of the above design is that any given object can only have a single region that may be exposed to user space. In cases where it is necessary to copy more than one field, those fields must be grouped together so that the single region covers them all. To get there, the patch set ends up reorganizing a few structures before whitelisting them. A dozen or so structures have been specifically whitelisted in the patch set. 

The final step in the patch set creates a new `GFP_USERCOPY` flag for memory allocations. There are certain system calls that can be used to force the kernel to allocate structures with a size controlled from user space. That is normally harmless, as long as the size kept within reasonable bounds. But certain types of attacks can benefit from the ability to create objects of a specific size. If those allocations are marked with `GFP_USERCOPY`, they will be taken from a separate slab, making it harder to control the layout of parts of the heap area. 

It's not clear when these patches will be pushed toward the mainline, but there do not appear to be any serious obstacles in their way.
