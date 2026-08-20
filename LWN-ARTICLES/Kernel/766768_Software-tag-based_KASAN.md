---
title: "Software-tag-based KASAN"
url: https://lwn.net/Articles/766768/
date: "September 26, 2018"
category: "Development tools-Sanitizers; KASan"
author: "By Jonathan Corbet September 26, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 26, 2018

The [kernel address sanitizer](<https://lwn.net/Articles/612153/>) (KASAN) is a kernel debugging tool meant to catch incorrect use of kernel pointers. It is an effective tool, if the number of KASAN-based bug reports showing up on the mailing lists is any indication. The downside of KASAN is a significant increase in the amount of memory used by a running system. The [software-tag-based mode](<https://lwn.net/ml/linux-kernel/cover.1537542735.git.andreyknvl@google.com/>) proposed by Andrey Konovalov has the potential to address that problem, but it brings some limitations of its own. 

KASAN works by allocating a shadow memory map to describe the addressability of the kernel's virtual address space. Each byte in the shadow map corresponds to eight bytes of address space and indicates how many of those eight bytes (if any) are currently accessible to the kernel. When the kernel allocates or frees a range of memory, the shadow map is updated accordingly. Using some instrumentation inserted by the compiler, KASAN checks each kernel pointer dereference against the shadow map to ensure that the kernel is meant to be accessing the pointed-to memory. If the shadow map indicates a problem, an error is raised. 

It is an effective technique and, thanks to the support from the compiler, the run-time CPU overhead is tolerable in many settings. But the shadow map requires a great deal of memory, and that does affect the usability of KASAN in the real world, especially when it is used on memory-constrained systems. This overhead is particularly painful for users who would like to run KASAN on production systems as an additional security measure. 

The new mode uses a different approach that takes advantage of an ARM64 feature called [top-byte ignore](<http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.den0024a/ch12s05s01.html>) (TBI). A 64-bit pointer allows for a large address space, rather larger than is actually needed on current systems, even if a web browser is running. When TBI is enabled, the system's memory-management unit will ignore the top byte of any address, allowing that byte to be used to store eight bits of arbitrary information. One possible use for that byte is to ensure that pointers into memory are pointing where they were intended to. 

In the software-tag-based mode, KASAN still allocates the memory map, but with some changes. Each byte in the map now corresponds to 16 bytes of real memory rather than eight, cutting the size of the map in half. Whenever the kernel allocates memory, a random, eight-bit tag value will be chosen. The pointer to the allocated object (which is aligned to a 16-byte boundary) will have that tag value set in the top byte; the tag value is also stored into the shadow memory map at the location(s) corresponding to that object. Whenever the returned pointer is dereferenced, its embedded tag value will be compared (using instrumentation from the compiler again) against the tag stored in the shadow memory map; if they do not match, an error will be logged. 

There are some clear advantages to this mode, starting with the halving of the amount of memory required for the shadow map. Current KASAN can only catch references to memory that the kernel is not meant to access at all; the new mode can catch the use of pointers that have strayed into the wrong part of kernel memory. On the other hand, the new mode will fail to catch a reference just beyond an allocated object if it falls within the 16-byte resolution of the map. There is a small possibility that an errant pointer will hit another region of memory that happened to get the same tag; such an access would not be detected. This mode will also only work on ARM64 processors, and it requires at least version 7 of the Clang compiler. 

There is another potential issue with the use of the software-tag-based mode. Address translation will ignore the top byte of a pointer when TBI is turned on, but other operations, such as pointer arithmetic and pointer comparisons, will not. Subtracting one pointer from another is a common operation in C programs; if those two pointers have different tag values, though, the result is unlikely to be what the developer intended. An erroneous subtraction is likely to make itself known quickly, but a comparison for equality that fails because two otherwise equal pointers have different tags could lead to rather more subtle problems. One can argue that pointers with different tags will have originated from different allocations and should not be compared anyway, but [worries](<https://lwn.net/ml/linux-kernel/20180628105057.GA26019@e103592.cambridge.arm.com/>) about the possibility of breaking things have led to some long discussions after previous postings of this work. 

In an attempt to address these concerns, Konovalov ran some extensive tests to try to find potential problems: 

All pointer comparisons/subtractions have been instrumented in an LLVM compiler pass and a kernel module that would print a bug report whenever two pointers with different tags are being compared/subtracted (ignoring comparisons with NULL pointers and with pointers obtained by casting an error code to a pointer type) has been used. 

The test turned up a number of places where such operations were taking place, but none of them turned out to be situations where the pointer tags changed the kernel's behavior; see the patch posting linked above for the full discussion. 

There is a small set of benchmark results included in the patch as well; it shows that software-tag-based KASAN performs similarly to regular KASAN in terms of CPU usage, though network bandwidth does drop somewhat. The new mode does use quite a bit less memory, though, as expected. KASAN remains far from free in either mode, though, tripling the time required for the test system to boot and reducing the networking performance to less than half of what is otherwise possible. So it is still going to be hard to use KASAN in production systems most of the time. 

Upcoming technologies, such as Arm's [memory tagging](<https://community.arm.com/processors/b/blog/posts/arm-a-profile-architecture-2018-developments-armv85a>), promise to support much of this functionality in hardware, which may change the equation somewhat. For the time being, though, KASAN must be implemented in software. It has found a number of bugs in the kernel, and would certainly find more if it were able to run in more contexts. The software-tag-based mode should make it possible to use KASAN on systems where its memory overhead is currently prohibitive, and that seems like a good thing.
