---
title: Tangled up in threads
url: https://lwn.net/Articles/294642/
date: "August 19, 2008"
category: "Memory management-User-space layout; Scalability"
author: "By Jonathan Corbet August 19, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
August 19, 2008

Certain kinds of programmers are highly enamored with threads, to the point that they use large numbers of them in their applications. In fact, some applications create many thousands of threads. Happily for this kind of developer - and their users - thread creation on Linux is quite fast. At least, most of the time. A situation where that turned out not to be the case gives an interesting look at what can happen when scalability and historical baggage collide. 

A user named Pardo recently [noted](<https://lwn.net/Articles/294644/>) that, in some situations, thread creation time on x86_64 systems can slow significantly - as in, by about two orders of magnitude. He was observing thread creation rates of less than 100/second; at such rates, the term "quite fast" no longer applies. Happily, Pardo also did much of the work required to track down the problem, making its resolution quite a bit easier. 

The problem with thread creation is the allocation of the stack to be used by the new thread. This allocation, done with `mmap()`, requires locating a few pages' worth of space in the process's address range. Calls to `mmap()` can be quite frequent, so the low-level code which finds the address space for the new mapping is written to be quick. Normally, it remembers (in `mm->free_area_cache`) the address just past the end of the previous allocation, which is usually the beginning of a big hole in the address space. So allocating more space does not require any sort of search. 

The `mmap()` call which creates a thread's stack is special, though, in that it involves the obscure, Linux-specific `MAP_32BIT` flag. This flag causes the allocation to be constrained to the bottom 2GB of the virtual address space - meaning it should really have been called `MAP_31BIT` instead. Thread stacks are kept in lower memory for a historical reason: on some early 64-bit processors, context switches were faster if the stack address fit into 32 bits. An application involving thousands of threads cannot help being highly sensitive to context switch times, so this was an optimization worth making. 

The problem is that this kind of constrained allocation causes `mmap()` to forget about `mm->free_area_cache`; instead, it performs a linear search through all of the virtual memory areas (VMAs) in the process's address space. Each thread stack will require at least one VMA, so this search gets longer as more threads are created. 

Where things really go wrong, though, is when there is no longer room to allocate a stack in the bottom 2GB of memory. At that point, the `mmap()` call will return failure to user space, which must then retry the operation without the `MAP_32BIT` flag. Even worse, the first call will have reset `mm->free_area_cache`, so the retry operation must search through the entire list of VMAs a second time before it is able to find a suitable piece of address space. Unsurprisingly, things start to get really slow at that point. 

But the really sad thing is that the performance benefit which came from using 32-bit stack addresses no longer exists with contemporary processors. Whatever problem caused the context-switch slowdown for larger addresses has long since been fixed. So this particular performance optimization would appear to have become something other than optimal. 

The solution which comes immediately to mind is to simply ignore the `MAP_32BIT` flag altogether. That approach would require that people experiencing this problem install a new kernel, but it would be painless beyond that. Unfortunately, nobody really knows for sure when the performance penalty for large stack addresses went away or how many still-deployed systems might be hurt by removing the `MAP_32BIT` behavior. So Andi Kleen, who first implemented this behavior, has [argued against its removal](<https://lwn.net/Articles/294649/>). He also points out that larger addresses could thwart a "pointer compression" optimization used by some Java virtual machine implementations. Andi would rather see the linear search through VMAs turned into something smarter. 

In the end, `MAP_32BIT` will remain, but the allocation of thread stacks in lower memory is going away anyway. Ingo Molnar has merged [a single-line patch](<https://lwn.net/Articles/294651/>) creating a new `mmap()` flag called `MAP_STACK`. This flag is defined as requesting a memory range which is suitable for use as a thread stack, but, in fact, it does not actually do anything. Ulrich Drepper will cause glibc to use this new flag as of the next release. The end result is that, once a user system has a new glibc and a fixed kernel, the old stack behavior will go away and that particular performance problem will be history. 

Given this outcome, why not just ignore `MAP_32BIT` in the kernel and avoid the need for a C library upgrade? `MAP_32BIT` is part of the user-space ABI, and nobody really knows how somebody might be using it. Breaking the ABI is not an option, so the old behavior must remain. On the other hand, one could argue for simply removing the use of `MAP_32BIT` in the creation of thread stacks, making the kernel upgrade unnecessary. As it happens, switching to `MAP_STACK` will have the same effect; older kernels, which do not recognize that flag, will simply ignore it. But if, at some future point, it turns out there still is a performance problem with higher-memory stacks on real systems, the kernel can be tweaked to implement the older behavior when it's running on an affected processor. So, with luck, all the bases are covered and this particular issue will not come back again.
