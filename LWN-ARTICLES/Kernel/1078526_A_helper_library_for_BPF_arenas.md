---
title: A helper library for BPF arenas
url: https://lwn.net/Articles/1078526/
date: "June 24, 2026"
category: BPF
author: "By Daroc Alden June 24, 2026 LSFMM+BPF"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Daroc Alden**  
June 24, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

[ BPF arenas](<https://lwn.net/Articles/961941/>) are areas of memory (potentially shared with user space) where programs have free reign to build their own data structures, unburdened by the verifier's bounds checks. Many of those data structures are potentially usable in multiple programs. Emil Tsalapatis brought his work on libarena, a library containing generic utilities for use in BPF arenas, to the 2026 [ Linux Storage, Filesystem, Memory-Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). Although the library is already available as part of the kernel, it is still in its early stages and he has more work planned. 

Tsalapatis works on sched_ext, a project that has seen him write a number of components based on BPF arenas that he believes could be reused. In particular, he sees potential for having a universally agreed-upon memory allocator for arenas, and a set of common data structures that use it. He also wants to see about adding better debugging capabilities to arenas; arenas may obviate the need to fight with the verifier, but that also means that there can be ordinary memory-safety problems that, while not a threat to the kernel, still need to be found and fixed. 

[ ![\[Emil Tsalapatis\]](https://static.lwn.net/images/2026/emil-tsalapatis-lsfmmbpf-small.png) ](<https://lwn.net/Articles/1078966>)

There are downsides to using arenas for everything, Tsalapatis admitted. Since BPF programs can write to arenas at any time, unchecked, the verifier cannot trust pointers to kernel objects stored in an arena to remain valid. Therefore, the verifier bans storing pointers to kernel objects in arenas at all; those pointers still need to be stored in other types of BPF map. Even with that caveat, he sees focusing on arenas as being worth the hassle. The end goal, he says, is to make BPF C as easy to write as normal C, and arenas are a big part of that. 

He envisions libarena as providing a standard library for BPF programs. It would be plain C code that is compiled and linked into BPF programs in the normal way, not an additional kernel interface to maintain. That also means it is optional — small BPF programs would not need to link against it unless they used its features. The library source code would live directly in the kernel tree, and be tested against the verifier to make sure the algorithms and data structures it provides do verify. Since libarena corresponds to a kernel version, it can also make use of the latest verifier features without worry. 

One member of the audience questioned how libarena would support migrating programs across kernel versions. BPF programs that use kernel interfaces can use "compile once — run everywhere" (CO-RE) relocations for that purpose, but these wouldn't work with code that is actually linked into BPF programs. Tsalapatis said that libarena wouldn't be supported on kernel versions prior to its introduction, but that with some care it could be made forward-compatible, so that people can build against the libarena version from the oldest kernel version that they wish to target, with no need for CO-RE relocations. 

The questioner was doubtful; they are currently trying to fix a problem where a program verifies on the 6.9 kernel but not the 6.19 kernel, so forward compatibility is a real problem. Perhaps programs could link in the version of libarena corresponding to the version of the kernel in use, Tsalapatis suggested. The hope is to have a stable enough interface that linking different versions for different kernels should not cause problems. 

He then explained how he envisioned programmers making use of the library. The source lives in the kernel tree, but he wants to periodically export it to a separate Git repository so that users can add it as a self-contained Git submodule, the same way that libbpf works. The build system would call into libarena's makefile to produce `libarena.bpf.o`. The main BPF program would include libarena's headers and link against the object. 

Another audience member asked whether this would require old kernels to be added to BPF's continuous-integration testing, but Tsalapatis reiterated that backward compatibility was not a priority. Alexei Starovoitov suggested worrying about that for the 7.5 kernel, noting that libarena was in the early stages of development. 

As of the 7.2 kernel, libarena will be available for experimental use. It contains a buddy allocator and some basic scaffolding for future work, but no actual data-structure implementations yet. It also has the hooks needed to work with Clang's address sanitizer, ASAN, although only with the in-development version of Clang. The allocator is usable, handling all sizes of allocation, but not particularly fast yet. Tsalapatis found writing the allocator to be a good case study for arenas, uncovering a lot of edge cases that needed to be addressed. 

In the future, libarena's allocator might end up using the design of [ mimalloc](<https://github.com/microsoft/mimalloc#mimalloc>) instead of a plain buddy allocator. Mimalloc was designed for use with functional-language runtimes, and performs well with multithreading and short-lived allocations, two features that Tsalapatis expects to be important for performance in the BPF programs of the future. It's also a simple and efficient design, he said. 

Previously, he had experimented with using a slab allocator for its reduced fragmentation, but he found it clunky for arbitrary programs. For sched_ext, it worked well because there were only ""like three types"" of structure needed, but that's not true of typical programs. 

In the future, Tsalapatis would like to offer red-black trees, B-trees, bitmaps, and [ Lev-Chase queues](<https://www.dre.vanderbilt.edu/~schmidt/PDF/work-stealing-dequeue.pdf>) (also known as Chase-Lev queues) as part of libarena at a minimum. Even if people start using large language models to write BPF programs, he thought there was value in having manually coded data structures in libarena that worked with the verifier. ""In my experience, I've found that seeding agents with good code lets them one-shot hard problems."" 

He then went over what the ASAN hooks do. When an arena is created, pages are mapped lazily — touching a new page causes a page fault. The BPF program can listen for these events and use them to also populate a metadata region containing ASAN's information about which areas of memory are okay to access. Allocating a section of memory marks it as safe with an eight-byte granularity, and freeing it removes the mark. The compiler augments pointer dereferences to check the state of the metadata before loading or storing a value. The work needs the in-development version of Clang because the compiler previously assumed that the metadata memory was always in address space zero, but BPF arenas use address space one, so Tsalapatis had to add a flag for that. (Clang numbers the available address spaces, while GCC [names them](<https://gcc.gnu.org/onlinedocs/gcc/Named-Address-Spaces.html>).) 

One audience member asked whether Tsalapatis had considered including code in libarena to navigate some of the kernel data structures that are difficult to navigate correctly. He had considered it, he said, but thought that it was better to keep the scope of libarena small for now, and focus on providing the core things that every program would need. At that point, time for the session ran out, but work on libarena has [ continued](<https://lwn.net/ml/all/20260618085626.19633-1-emil@etsalapatis.com/>) since the talk, with some of the planned-on data structures now available to use.
