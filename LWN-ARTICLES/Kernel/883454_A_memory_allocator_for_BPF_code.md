---
title: A memory allocator for BPF code
url: https://lwn.net/Articles/883454/
date: "February 4, 2022"
category: "BPF-Memory management; Memory management-BPF; Releases-5.18"
author: "By Jonathan Corbet February 4, 2022"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
February 4, 2022

Loading a BPF program into the kernel involves a lot of steps, including verification, permissions checking, linking to in-kernel helper functions, and compilation to the native instruction format. Underneath all of that, though, lies one other simple task: allocating some memory to store the compiled BPF program in the kernel's address space. It turns out that this allocation can be somewhat wasteful of memory in current kernels, especially on systems where large numbers of BPF programs are loaded. [This patch set](<https://lwn.net/ml/linux-kernel/20220201062803.2675204-1-song@kernel.org/>) from Song Liu seeks to remedy this problem by introducing yet another specialized memory allocator into the kernel. 

The kernel allows BPF programs to be reasonably large, but most of them are, in practice, quite small. BPF can do quite a bit with a few hundred bytes of code space. But the kernel allocates space for BPF programs in units of full base (usually 4KB) pages, with all of the space past the end of each program being wasted for as long as that program remains loaded. For a small program, most of the page allocated to hold the code is unused; if there are many programs loaded, the amount of wasted memory can start to add up. 

Liu's patch set adds a new "bpf_prog_pack" allocator to address this problem. It would be, on the surface, one of the simplest memory allocators in the system. It maintains a list of huge pages to hold BPF programs, allocating new ones as needed. Whenever space is needed for a new BPF program, a simple, first-fit algorithm allocates the amount needed. A bitmap associated with each huge page (or "pack") tracks the free space (in 64B chunks), thus automatically coalescing chunks returned to the allocator when programs are unloaded. It is not written for speed, but it does not need to be; even on a system that makes heavy use of BPF, allocation and free operations will be relatively rare. 

The advantages of this simple allocator are reasonably clear. Because multiple programs can be packed into a single page, the memory that is wasted due to internal fragmentation will be greatly reduced. Performance will be helped by putting BPF programs into huge pages, which reduces translation lookaside buffer (TLB) pressure. But it is worth asking why yet another allocator is needed when the kernel already has memory-management code that has been extensively developed and tuned over the years. The answer is that BPF brings some special needs that cannot be met with existing allocators. 

The problem with using the kernel's page allocator without a subdividing layer has already been described: too much space is wasted. But the kernel's slab allocators have been designed for exactly this task: they take large chunks from the page allocator and pass them out to the rest of the kernel in smaller pieces. So it might be natural to think about using the slab allocator here; Liu does not explain why that was not done, but there are a couple of plausible reasons for this choice. 

One reason is that the slab allocators are optimized for the allocation of equal-sized chunks. But there is no one size that fits all for BPF programs, so the simple expedient of using a dedicated slab and `kmem_cache_alloc()` will not work here. It would be possible to use multiple slabs, of course, either directly within the BPF code or simply by allocating space with `kmalloc()`, but there would be internal fragmentation issues there (albeit smaller) either way. The power-of-two approach used by `kmalloc()` would still waste a fair amount of space at the end of each program. 

There is a more fundamental problem with using the slab allocator, though. BPF programs are executable code, which must be handled specially. Code requires execute permissions at the page level, meaning that it cannot be intermixed with data (as `kmalloc()` would do). After all, developers have gone out of their way for years to ensure that data cannot be made executable within the kernel. So the slab allocators, which are meant to manage space for data, are not really suitable for this application. 

A certain amount of trickery is required to make the program-packing scheme work, even with the special-purpose allocator. Updating code in a running system is fraught with perils; if a CPU in the system tries to run code while it is being changed, the results tend to be bad. The kernel relies fairly heavily on self-modifying code, so the mechanisms to do this modification safely are there, but the task must still be handled carefully. The inability to write to executable memory in straightforward ways, along with the fact that executable memory in the kernel must be read-only, presents challenges for the JIT compiler. 

So when the new allocator is invoked to find a spot for a new program, it actually allocates _two_ buffers. The first is the read-only, executable space where the program is destined to live; the second is ordinary memory obtained with `kvmalloc()`. The JIT compiler will compile into the second buffer, which can be written to, but any addresses it generates must be relative to the first buffer instead. Once the compilation process is complete, the kernel's "text poke" machinery, [enhanced](<https://lwn.net/ml/linux-kernel/20220201062803.2675204-6-song@kernel.org/>) with a new copy function, can be used to move the compiled program into the read-only space and the temporary buffer freed. 

This patch set has been through eight revisions as of this writing, with a fair amount of review activity creating a stream of new items to fix. At this point, though, there may not be a whole lot more to change. So it would not be entirely surprising to see this code find its way into 5.18; heavy BPF users should benefit with reduced memory usage and a small performance improvement.
