---
title: The kernel address sanitizer
url: https://lwn.net/Articles/612153/
date: "September 17, 2014"
category: "Development tools-Sanitizers; KASan"
author: "By Jake Edge September 17, 2014"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
September 17, 2014

Finding places where the kernel accesses memory that it shouldn't is the goal for the kernel address sanitizer (KASan). It uses a combination of a new GCC feature to instrument memory accesses and "shadow memory" to track which addresses are legitimate, so that it can complain loudly when the kernel reads or writes anywhere else. While KASan shares some attributes with other kernel debugging features, it has its own advantages—and has already been used to find real kernel bugs. 

The [KASan patch set](<https://lwn.net/Articles/611410/>) comes from Andrey Ryabinin, though it is based on work by Andrey Konovalov and by the [AddressSanitizer project](<https://code.google.com/p/address-sanitizer/>). While the patches are still at the RFC stage, this is the second version of them. Many improvements have been made since the [original RFC](<https://lwn.net/Articles/604817/>) was posted back in July. 

The basic idea behind KASan is to allocate a map (aka "shadow region") that represents each eight bytes of kernel address space with one byte in the map. For x86_64 (which is the only architecture supported), that means setting aside (but not allocating) 16TB of address space to handle the entire 128TB that the kernel can address. Each byte in the map encodes the legality of a kernel access to the corresponding bytes in the full address space. 

The encoding is fairly straightforward. A 0 means that all eight bytes are legal for access, while 1-7 indicate the number of consecutive bytes at the beginning of the eight-byte region that are valid (so 2 means that the first two bytes are valid, the last six are not). Negative values are for different types of non-accessible memory (free memory, redzones of various sorts, etc.). It would seem that other positive values could be used to encode consecutive valid bytes at the end of the eight-byte region, but perhaps that does not occur in practice. 

Kernel memory accesses can then be checked against the shadow map to see if they are valid. GCC 5.0 (the next major release after 4.9, which is expected in 2015) will introduce a new feature that allows runtime checking of kernel addresses. The `-fsanitize=kernel-address` flag will cause GCC to instrument each memory load and store operation. It will insert a function call into each of those operations that can examine the target address and complain into the kernel log if it is invalid. The calls are of the form `__asan_loadN()` and `__asan_storeN()`, where `N` corresponds to the width of the access (1, 2, 4, 8, or 16 bytes). There are GCC patches available to turn these out-of-line checks into inline checks, where GCC will directly insert the code to check the shadow map, rather than making function calls to do so. 

So, the bulk of the work is done by the code inserted by the compiler. But there are still some details to be handled by the patches. Implementing the checking and reporting infrastructure is [step one](<https://lwn.net/Articles/612266/>) (that patch also includes a `Documentation/kasan.txt` file). Then, the initial shadow region needs to be populated. Early in the boot process, each page table entry in the shadow region is set to the zero page. Later, when the physical memory has been mapped, the zero pages corresponding to that memory are unmapped, and real pages are allocated and mapped for tracking the kernel's memory. 

As pages are allocated by the kernel, they are marked as accessible in the shadow region; likewise, as pages are freed, they are marked inaccessible. The SLUB allocator has been modified to update the shadow map for its allocations and deallocations. 

In the patch set's first message, Ryabinin outlined the differences between KASan and a few other kernel memory-debugging tools. Since KASan uses compile-time instrumentation, it is much faster than [kmemcheck](<https://www.kernel.org/doc/Documentation/kmemcheck.txt>), but it cannot detect reads of uninitialized memory as kmemcheck does. While both `DEBUG_PAGEALLOC` and `SLUB_DEBUG` are faster than KASan, neither can detect all of the illegal accesses that KASan does (`DEBUG_PAGEALLOC` only has page-level granularity and `SLUB_DEBUG` is unable to detect some bad reads). 

KASan is enabled for the kernel with the `KASAN` Kconfig parameter. It requires the SLUB allocator and a GCC version >= 5.0. Better reports will be generated if stack traces and `SLUB_DEBUG` are enabled. Ryabinin has also added a [test module](<https://lwn.net/Articles/612265/>) that can be used to cause out-of-bounds accesses and use-after-free errors for testing KASan or other memory debuggers. 

The discussion on the patches has been constructive, with just a few suggestions for relatively minor changes. In addition, Sasha Levin [listed several kernel bugs](<https://lwn.net/Articles/612247/>) that he had found using the [Trinity fuzzer](<https://lwn.net/Articles/536173/>) with the first version of the KASan patch set. It would seem that there is an unfilled niche in the kernel's memory debugging that KASan could help fill.
