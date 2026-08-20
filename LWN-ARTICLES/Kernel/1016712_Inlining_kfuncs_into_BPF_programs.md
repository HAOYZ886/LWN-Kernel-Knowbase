---
title: Inlining kfuncs into BPF programs
url: https://lwn.net/Articles/1016712/
date: "April 11, 2025"
category: "BPF-kfuncs"
author: "By Daroc Alden April 11, 2025 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Daroc Alden**  
April 11, 2025

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2025/>)

Eduard Zingerman presented a daring proposal that ""makes sense if you think about it a bit"" at the 2025 Linux Storage, Filesystem, Memory-Management, and BPF Summit. He wants to inline performance-sensitive kernel functions into the BPF programs that call them. His prototype does not yet address all of the design problems inherent in that idea, but it did spark a lengthy discussion about the feasibility of his proposal. 

The justification for inlining, as always, is optimization. The BPF verifier's primary role is to analyze the safety of BPF programs. But it does also use the information learned during that analysis to eliminate unnecessary bounds-checks at run time. The same information could potentially eliminate conditional branches in kfuncs (kernel functions callable from BPF) that are used by frequently-invoked BPF programs. 

Zingerman first [proposed](<https://lwn.net/Articles/997401/>) kfunc inlining in November 2024. In his initial request for comments, he focused on [ `bpf_dynptr_slice()`](<https://elixir.bootlin.com/linux/v6.13.7/source/kernel/bpf/helpers.c#L2557>) and showed that inlining it could eliminate jumps from a switch statement and an if statement, providing a 1.53x speed up on his synthetic benchmark. 

`bpf_dynptr_slice()` is 40 lines of code with 10 conditionals spread across the function, Zingerman said, so inlining it by hand was ""a bit tedious"". An automated solution that inlines specific kfuncs that benefit from the transformation could potentially be useful. 

To do that, he proposed compiling those functions to BPF and embedding the resulting BPF code into the kernel binary. The verifier could inline the code into loaded BPF programs during verification. The problem with that approach is that, currently, BPF is not architecture specific — the same programs will run unmodified on any architecture that supports BPF — but kfuncs can have architecture-specific code. In fact, many kernel headers have architecture-specific functions that need to be taken into account. 

One potential workaround could be to compile the kfuncs for the host architecture, but then use some LLVM-specific tooling to retarget the resulting binary to BPF. Another solution could be to only support inlining functions (such as `bpf_dynptr_slice()`) that don't contain architecture-specific code. Alternatively, any architecture-specific code could be pulled out into a separate kernel function callable from BPF, and the remaining code could be compiled directly. Zingerman didn't really seem happy with any of those approaches, though. 

José Marchesi questioned why it was necessary to build for a particular architecture and then retarget it to BPF in the first place; after all, any architecture-specific assembly code wouldn't be able to be retargeted. Zingerman explained that firstly, the build would fail because the kernel doesn't have a "BPF" architecture in its build system, and secondly, there are some data structures that vary in layout depending on the architecture which need to match. 

Andrii Nakryiko pointed out that 32-bit architectures need 32-bit pointers, at least, even if more complex structures could be made to work somehow. Marchesi conceded the point, but suggested that the compiler could pretend to be the host architecture to the preprocessor, and still emit BPF code. Alexei Starovoitov explained that there were parts of the build system that would need to be adapted too. Zingerman summarized the situation by saying that they could try it, but there would be obstacles. 

I suggested that having multiple architectures (BPF+x86, BPF+risc, etc.) might help. Zingerman agreed that, in the future, it might come to that. But, for his initial proposal, the workaround with LLVM works well enough to prove out the concept. 

He was more certain about the right approach for embedding any generated BPF into the kernel: adding them to a data section and using the ELF symbol table to find them when needed. The verifier will need to handle applying relocations inside the kfuncs' bodies, which will allow them to call other functions in the kernel transparently. 

The inlining itself is also fairly simple: before verifying the user's program, the verifier should make a copy of each inlinable kfunc for each call in the program. When it reaches a call to the function during verification, it sets up a special verifier state to pass information about the arguments and BPF stack into the code that verifies the instance of the function. Then it verifies the body of the kfunc, ideally using its contextual information to do dead-code elimination. When the BPF code is compiled, the body of the kfunc can be directly included by the just-in-time compiler. 

Having a separate instance of the kfunc for each call is not strictly necessary, Zingerman said, but doing it that way keeps the impact on the verifier minimal. To share verification of one kfunc body between call sites, the verifier would need to track additional information about the logical program stack and actual program stack that it does not currently handle. That representation would be complex and harder to reason about, he explained, which is why he favored the more isolated approach. 

All of this discussion of the mechanism behind inlining was predicated on being able to choose which functions would benefit from inlining, however. Currently, Zingerman is considering focusing on functions for manipulating [ dynptrs](<https://lwn.net/Articles/910873/>) and some iterator functions, although he's open to expanding the set of inlinable kernel functions over time. 

Nakryiko asked about how Zingerman intended to check the types of arguments passed to an inlined kfunc during verification. For the initial version, he didn't worry about that, Zingerman said. He just assumed that the kernel function was compiled correctly. But in the future, the build process could embed BTF debugging information alongside the compiled kfunc and it could be checked that way. 

Arnaldo Carvalho de Melo wanted to know about conditional inlining — that is, inlining only the calls to kfuncs that are most used. Starovoitov replied that the BPF subsystem does not currently have any kind of profile-guided optimization. Zingerman said that it was another thing to be explored, but that it wasn't part of his initial proposal. Starovoitov suggested tracking which branches are taken at run time, and reoptimizing the BPF program on the fly. ""We aren't doing that, but it would be cool,"" he said. 

Nakryiko also wanted to know why this kind of inlining needed to be done automatically. Zingerman said that making it automatic ensures that inlining doesn't introduce any mistakes, and can be done for more complex functions. Nakryiko suggested that it doesn't really make sense to inline something that's already complicated. Zingerman agreed, saying that was one reason he wanted to lead a session at the summit — to see which other functions, beyond the few he had focused on, people were interested in inlining. 

Daniel Borkmann, one of the organizers for the BPF track, suggested that it would be interesting to evaluate the impact of inlining some functions for handling BPF maps. But then he advised people that the session had run out of time, and brought things to a close there.
