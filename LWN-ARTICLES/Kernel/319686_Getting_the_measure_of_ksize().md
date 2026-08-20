---
title: Getting the measure of ksize()
url: https://lwn.net/Articles/319686/
date: "February 17, 2009"
category: "Memory management-Internal API; ksize"
author: "By Jonathan Corbet February 17, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 17, 2009

One of the lesser-known functions supported by the kernel's memory management code is `ksize()`; given a pointer to an object allocated with `kmalloc()`, `ksize()` will return the size of that object. This function is not often needed; callers to `kmalloc()` usually know what they allocated. It can be useful, though, in situations where a function needs to know the size of an object and does not have that information handy. As it happens, there are other potential uses for `ksize()`, but there are traps as well. 

Users of `ksize()` in the mainline kernel are rare. Until 2008, the main user was the nommu architecture code, which was [found to be using `ksize()` in a number of situations](<http://lkml.org/lkml/2008/5/22/226>) where that use was not appropriate. The result was a cleanup of the nommu code and the un-exporting of `ksize()` in an attempt to prevent that sort of situation from coming about again. 

Happiness prevailed until recently; the 2.6.29-rc5 kernel includes [a patch to the crypto code](<https://lwn.net/Articles/319689/>) which makes use of `ksize()` to ensure that `crypto_tfm` structures are completely wiped of sensitive data before being returned to the system. The lack of an export for `ksize()` caused the crypto code to fail when built as a module, so Kirill Shutemov posted [a patch to export it](<https://lwn.net/Articles/319690/>). That's when the discussion got interesting. 

There was resistance to restoring the export for `ksize()`; the biggest problem would appear to be that it's an easy function to use incorrectly. It is only really correct to call `ksize()` with a pointer obtained from `kmalloc()`, but programmers seem to find themselves tempted to use it on other types of objects as well. This situation is not helped by the fact that the SLAB and SLUB memory allocators work just fine if any slab-allocated memory object is passed to `ksize()`. The SLOB allocator, instead, is not so accommodating. An explanation of this situation led to [some complaints](<https://lwn.net/Articles/319691/>) from Andrew Morton: 

OK. This is really bad, isn't it? People will write code which happily works under slab and slub, only to have it crash for those small number of people who (very much later) test with slob? 

[...] 

Gee this sucks. Biggest mistake I ever made. Are we working hard enough to remove some of these sl?b implementations? Would it help if I randomly deleted a couple? 

Thus far, no implementations have been deleted; indeed, it appears that the [SLQB allocator](<http://lwn.net/Articles/311502/>) is headed for inclusion in 2.6.30. The idea of restricting access to `ksize()` has also not gotten very far; the export of this function was restored for 2.6.29-rc5. In the end, the kernel is full of dangerous functions - such is the nature of kernel code - and it is not possible to defend against any mistake which could be made by kernel developers. As Matt Mackall [put it](<https://lwn.net/Articles/319698/>), this is just another basic mistake: 

And it -is- a category error. The fact that kmalloc is implemented on top of kmem_cache_alloc is an implementation detail that callers should not assume. They shouldn't call kfree() on kmem_cache_alloc objects (even though it might just happen to work), nor should they call ksize(). 

There is another potential reason to keep this function available: `ksize()` may prove to have a use beyond freeing developers from the need to track the size of allocated objects. One poorly-kept secret about `kmalloc()` is that it tends to allocate objects which are larger than the caller requests. A quick look at `/proc/slabinfo` will (with the right memory allocator) reveal a number of caches with names like `kmalloc-256`. Whenever a call to `kmalloc()` is made, the requested size will be rounded up to the next slab size, and an object of that size will be returned. (Again, this is true for the SLAB and SLUB allocators; SLOB is a special case). 

This rounding-up results in a simpler and faster allocator, but those benefits are gained at the cost of some wasted memory. That is one of the reasons why it makes sense to create a dedicated slab for frequently-allocated objects. There is one interesting allocation case which is stuck with `kmalloc()`, though, for DMA-compatibility reasons: SKB (network packet buffer) allocations. 

An SKB is typically sized to match the maximum transfer size for the intended network interface. In an Ethernet-dominated world, that size tends to be 1500 bytes. A 1500-byte object requested from `kmalloc()` will typically result in the allocation a 2048-byte chunk of memory; that's a significant amount of wasted RAM. As it happens, though, the network developers really need the SKB buffer to not cross page boundaries, so there is generally no way to avoid that waste. 

But there may be a way to take advantage of it. Occasionally, the network layer needs to store some extra data associated with a packet; IPSec, it seems, is especially likely to create this type of situation. The networking layer could allocate more memory for that data, or it could use `krealloc()` to expand the existing buffer allocation, but both will slow down the highly-tuned networking core. What would be a lot nicer would be to just use some extra space that happened to be lying around. With a buffer from `kmalloc()`, that space might just be there. The way to find out, of course, is to use `ksize()`. And that's exactly what the networking developers intend to do. 

Not everybody is convinced that this kind of trick is worth the trouble. Some argue that the extra space should be allocated explicitly if it will be needed later. Others would like to see some benchmarks demonstrating that there is a real-world benefit from this technique. But, in the end, kernel developers do appreciate a good trick. So `ksize()` will be there should this kind of code head for the mainline in the future.
