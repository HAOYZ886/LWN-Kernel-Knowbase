---
title: When memory allocation failure is not an option
url: https://lwn.net/Articles/401915/
date: "August 25, 2010"
category: "Memory management-Internal API"
author: "By Jonathan Corbet August 25, 2010"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 25, 2010

One of the darker corners of the kernel's memory management subsystem is the `__GFP_NOFAIL` flag. That flag says that an allocation request cannot be allowed to fail regardless of whether memory is actually available or not; if the request cannot be satisfied, the allocator will loop continuously in the hope that, somehow, something will eventually change. Needless to say, kernel developers are not encouraged to use this option. More recently, David Rientjes has been trying [to get rid of it](<https://lwn.net/Articles/401917/>) by pushing the (possibly) infinite looping back to callers. 

Andrew Morton [was not convinced](<https://lwn.net/Articles/401918/>) by the patch: 

The reason for adding GFP_NOFAIL in the first place was my observation that the kernel contained lots of open-coded retry-for-ever loops. 

All of these are wrong, bad, buggy and mustfix. So we consolidated the wrongbadbuggymustfix concept into the core MM so that miscreants could be easily identified and hopefully fixed. 

David's response is that said miscreants have not been fixed over the course of many years, and that `__GFP_NOFAIL` imposes complexity on the page allocator which slows things down for all users. Andrew came back with a suggestion for special versions of the allocation functions which would perform the looping; that would move the implementation out of the core allocator, but still make it possible to search for code needing to fix; David obliged with [a patch](<http://lwn.net/Articles/401677/>) adding `kmalloc_nofail()` and friends. 

This kind of patch is guaranteed to bring out comments from those who feel that it is far better to just fix code which is not prepared to deal with memory allocation failures. But, as Ted Ts'o [pointed out](<https://lwn.net/Articles/401919/>), that is not always an easy thing to do: 

So we can mark the retry loop helper function as deprecated, and that will make some of these cases go away, but ultimately if we're going to fail the memory allocation, something bad is going to happen, and the only question is whether we want to have something bad happen by looping in the memory allocator, or to force the file system to panic/oops the system, or have random application die and/or lose user data because they don't expect write() to return ENOMEM. 

Ted's point is that there are always going to be places where recovery from a memory allocation failure is quite hard, if it's possible at all. So the kernel can provide some means by which looping on failure can be done centrally, or see it done in various ad hoc ways in random places in the kernel. Bad code is not improved by being swept under the rug, so it seems likely that some sort of central loop-on-failure mechanism will continue to exist indefinitely.
