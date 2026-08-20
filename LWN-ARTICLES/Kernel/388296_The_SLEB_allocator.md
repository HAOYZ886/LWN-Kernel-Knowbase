---
title: The SLEB allocator
url: https://lwn.net/Articles/388296/
date: "May 19, 2010"
category: "Memory management-Slab allocators; Slab allocator"
author: "By Jonathan Corbet May 19, 2010"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 19, 2010

Longtime LWN readers will know that the kernel does not have just one internal memory allocator. Instead, we have the longstanding "slab" allocator (perennially due to be removed someday), the SLUB allocator (intended for better scalability, but it hasn't been able to beat slab on every test), and the SLOB allocator (a space-efficient allocator for embedded use). There is also the [SLQB](<http://lwn.net/Articles/311502/>) allocator waiting in the wings, but it has been waiting there long enough that one may wonder if it will ever emerge from there. 

All told, one might assume that we have enough allocators. Then again, there are quite a few letters still available in the SL*B namespace, so why not make another one? 

Thus Christoph Lameter, author of SLUB, has come forward with the [SLEB allocator](<http://lwn.net/Articles/387924/>), which is meant to be a mixture of the best of slab and SLUB. Unlike SLUB, SLEB retains the object queues used by slab, but it also adds a bitmap for object management as well. Also unlike SLUB, there is no storage of metadata in the objects themselves. That is a performance enhancement: if a cache-cold object is allocated or freed, SLEB will not bring it into the cache. 

This code is very new; it apparently has not yet been trusted outside of a KVM virtual machine. The long benchmarking process that might lead to merging and, possibly, displacing one of the other allocators has not yet begun. But the code is there, and that's a start.
