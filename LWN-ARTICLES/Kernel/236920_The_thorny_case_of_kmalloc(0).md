---
title: The thorny case of kmalloc(0)
url: https://lwn.net/Articles/236920/
date: "June 5, 2007"
category: "Development model-Kernel quality; Memory management-Internal API"
author: 
---

People running 2.6.22-rc kernels have likely noticed the occasional warning and traceback associated with zero-length allocations. It turns out that there is code in the kernel which asks `kmalloc()` to allocate a zero-sized object. Nobody really knew how often this happens until the warning went in as part of the [SLUB allocator](<http://lwn.net/Articles/229984/>) patch set; now that these cases are turning up, it seems that deciding what to do about them is harder than one might expect. 

One possibility is to return `NULL`. On the face of it, this option would appear to make sense; the caller has requested that no memory be allocated, and `kmalloc()` has complied. The problem here is that a `NULL` pointer is already loaded with meaning. It says that the allocation has failed (which it didn't - there is always enough memory left to allocate another zero bytes) and is often used as an indicator that a particular structure or subsystem has not been initialized. More to the point, it seems that there is an occasional situation where a zero-length allocation is not entirely incorrect; consider the allocation of a structure which, as a result of the kernel's configuration options, has been optimized down to zero members. Coding around such cases is possible, but it is not clear that adding more twists and turns is worth the trouble when zero-length allocations can just be handled in `kmalloc()`. 

Another possibility is to return the smallest object that `kmalloc()` can manage - currently eight bytes. That is what `kmalloc()` has silently done for years. This solution appears to work, but it has the disadvantage of returning memory which can be written to. A zero-length allocation can arguably be correct, but it's hard to find anybody who would agree that storing data into a zero-length chunk of memory makes sense. Even highly compressed data cannot be expected to fit into that space in all situations. People who worry about finding bugs would much prefer that any attempt to actually write to memory allocated with `kmalloc(0)` caused the kernel to protest in a very noisy way. 

That brings us to the third possibility: [this patch from Christoph Lameter](<http://lwn.net/Articles/236809/>) which causes `kmalloc(0)` to return a special `ZERO_SIZE_PTR` value. It is a non-`NULL` value which looks like a legitimate pointer, but which causes a fault on any attempt at dereferencing it. Any attempt to call `kfree()` with this special value will do the right thing, of course. 

The final option seems like it should be the right course, allowing zero-length allocations without masking any subsequent incorrect behavior. Surprisingly, though, there is an objection here too: now every call to `kmalloc(0)` returns the same value. One might not think this would be a problem; subsequent zero-length allocations will all be zero bytes apart, just like the C standard says they should be. But some developers are worried that this behavior might confuse code which compares pointers to see if two objects are the same. There is also, apparently, an established coding pattern (in user space) which uses zero-length allocations as a way of generating a unique cookie value. If all zero-length allocations return the same pointer, these cookies lose their uniqueness. 

That worry appears unlikely to carry the day, though; [Linus says](<https://lwn.net/Articles/236921/>): 

If people can't be bothered to create a "random ID generator" themselves, they had damn well better use "kmalloc(1)" rather than "kmalloc(0)" to get a unique cookie. Asking the allocator to do something idiotic because some idiot thinks a memory allocator is a cookie allocator is just crazy. 

I can understand that things like user-level libraries have to take crazy people into account, but the kernel internal libraries definitely do not. 

Add to this argument the fact that nobody seems to have discovered such a use of `kmalloc()` in the kernel yet, and the "unique cookie" argument runs out of steam. So some form of the `ZERO_SIZE_PTR` patch, with the warning removed, will probably find its way into the mainline - but probably not before 2.6.23.
