---
title: Better debugging information for inlined kernel functions
url: https://lwn.net/Articles/1018475/
date: "April 30, 2025"
category: "BPF-Compiler support"
author: "By Daroc Alden April 30, 2025 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Daroc Alden**  
April 30, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Modern compilers perform a lot of optimizations, which can complicate debugging. Song Liu and Thierry Treyer spoke about a potential improvement to [ BPF Type Format](<https://www.kernel.org/doc/html/latest/bpf/btf.html>) (BTF) debugging information that could partially combat that problem at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. They want to add information on selectively inlined functions to BTF in order to better support tracing tools. Treyer participated remotely. 

One of the most common compiler optimizations is inlining, which embeds the code of a function directly into its caller, avoiding the overhead of a function call and potentially exposing other hidden optimization opportunities. Modern compilers don't always inline every call to a particular function that would benefit from inlining, however. The compiler uses heuristics to decide which call sites will benefit from inlining. This means that a programmer can easily end up with a situation where a function still appears in a binary's symbol table (because some calls were not inlined), but tracing that function won't show calls to it (because the hot calls were inlined, and therefore the function's symbol no longer referrs to them). 

[ ![\[Song Liu\]](https://static.lwn.net/images/2025/song-liu-lsfmmbf-small.png) ](<https://lwn.net/Articles/1018490>)

That is a problem, Liu said. Both because it makes debugging harder, but also because it motivates developers to mark important kernel functions as not inlinable, so that they can rely on being able to trace them. It is technically possible to trace selectively inlined functions by finding the inlined locations using DWARF debugging information, which some tracing tools do automatically. DWARF is a bulky, complex format, however, which makes using it for this purpose slow. Other methods, such as tracepoints and Linux security-module hooks, also work with selectively inlined functions. Liu argued that those aren't a proper replacement for normal tracing, however, since when debugging a kernel problem it is often not clear what functions one will want to trace until one is actually working on the problem. 

Liu outlined two different options for how to improve this situation: just marking selectively inlined functions in the kernel's BTF, and including information about where they have been inlined. The first option has the benefit of simplicity; it would be easy to add an additional function attribute and convert [ tools like pahole](<https://lwn.net/Articles/1016243/>) to handle it. Selectively inlined functions are mostly a problem because of the confusion they cause, Liu said. Just being able to warn the developer about what is happening would help. 

The second idea would let other tools more easily match what `perf probe` does: set breakpoints at every location where a function was inlined, as well as its non-inlined location. `perf` does this by parsing DWARF; Liu's change would let other tools use BTF for the same purpose. 

Tracing an inlined function is not as simple as putting breakpoints in the right place, however. When a function is inlined, arguments and return values can disappear due to other optimizations. BTF would need a way to indicate how to transform the machine state at the callsite to recover the function's arguments. 

Liu and Treyer analyzed the kernel's existing debug information to figure out whether this was even possible. Out of 228,000 arguments to inlined functions, across 150,000 different call sites, the location of about 83% can be expressed as an offset from a register, usually because the argument is present on the calling function's stack or passed as a pointer. This analysis doesn't include a handful of the most commonly inlined functions, Liu warned, because those disproportionately skew the numbers. 

One audience member asked whether that number was per-argument or per-function. Liu clarified that it was per-argument. The audience member then asked what percentage of functions have only arguments representable in this way. Liu showed a slide indicating that approximately half of the functions he looked at would have all their arguments available using this scheme. 

With the knowledge that it was possible, Liu put together a proposal for how to encode information about inlined function arguments in BTF: each parameter would be described by a sequence of operators in a highly restricted virtual machine. Supported operations would include loading constants, dereferencing registers, applying offsets, etc. So, for example, an argument that was stored as a member of a structure pointed to by a register would be represented by something like "load register, add offset, dereference, end". Using Liu and Treyer's prototype implementation, this would add about 10MB to the kernel's BTF, although that number is before performing deduplication, which may help substantially. 

A final problem is that BTF only exists for functions that were not fully inlined. Functions that are inlined at every call site are less confusing than selectively inlined functions, but they would be fairly simple to support. Adding those in would add an additional 32,000 functions to the kernel's BTF. 

The audience had a number of questions about the design for encoding argument information, mostly focused around how to streamline it and remove redundant information. Alexei Starovoitov said that he thought this approach seemed ""inspired by the DWARF state machine"", and that they had the opportunity to do something simpler instead. Liu agreed that the proposal was a bit rough, and welcomed suggestions for how to improve it. 

Starovoitov also pointed out that the encoding Liu had proposed wouldn't work properly for 16-byte values, of which the kernel uses a fair number. Andrii Nakryiko asked whether the problem was that these values were sometimes stored in a pair of registers, instead of in memory. Starovoitov agreed that was sometimes the case, but outlined a few other edge-cases that small structures could fall into. Liu agreed to look into the encoding of 16-byte values.
