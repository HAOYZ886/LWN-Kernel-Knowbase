---
title: LLVM improvements for BPF verification
url: https://lwn.net/Articles/974945/
date: "May 27, 2024"
category: "BPF-Compiler support"
author: "By Daroc Alden May 27, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Daroc Alden**  
May 27, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Alan Jowett gave a remote presentation at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) about what features could be added to LLVM to make writing BPF programs easier. While there is nothing specific to LLVM about BPF code (and the next session in the track was led by GCC developer José Marchesi about better support for that compiler), LLVM is currently the most common way to turn C code into BPF bytecode. That translation, however, runs into problems when the BPF verifier cannot understand the code LLVM's optimizations produce. 

Jowett began by talking about how LLVM processes code internally. First, the C code is translated to [LLVM intermediate representation](<https://llvm.org/docs/LangRef.html>) (IR). Then, several passes of optimizations gradually turn the IR into a more efficient version. Finally, the code generator creates BPF bytecode corresponding to the IR. The problem with this process is that LLVM has had many years to develop sophisticated optimizations. It is not uncommon, Jowett said, for LLVM to produce code that is correct, but that the BPF verifier cannot understand — a problem [LWN has covered before](<https://lwn.net/Articles/946254>). For that reason, developers sometimes have to use inline BPF assembly to circumvent the optimizer in order to have their programs accepted. 

Before opening up discussion of possible solutions to the problem, however, Jowett first covered some other things that would be nice to see from LLVM. One example is likely/unlikely branch hints that may be present at the source level, but which are lost by the time a program is translated to bytecode. Another possibility Jowett raised was support for code-coverage information, possibly as a prelude to supporting profile-guided optimization of BPF programs. 

Jowett then presented a few rough ideas for how it might be possible to prevent LLVM optimization passes from breaking the verifiability of BPF code. One solution might be to move the verifier into the compiler, not as the authoritative source, but as a check to prevent optimization passes from making changes that the verifier cannot understand. Jowett did not propose using the kernel's BPF verifier, however — perhaps because of the licensing problems that would pose — but rather the [PREVAIL](<https://vbpf.github.io/>) verifier, an MIT-licensed verifier produced as an academic project that runs in polynomial time (as opposed to the kernel's exponential time). 

Using a different verifier is not a perfect solution, however. Jowett pointed out that PREVAIL's design means that it will not verify programs with "correlated branches", where taking one branch always implies that another branch should be taken as well. This is a somewhat common pattern in BPF programs that conditionally acquire and release locks, for example. This pattern can also be introduced artificially by the LLVM optimizer when it tries to avoid repeating tests. 

I asked Jowett how he planned to use the PREVAIL verifier inside LLVM's optimization passes, since the former operates on BPF bytecode and the latter operates on LLVM IR. Jowett acknowledged that it would be a problem. Marchesi noted that something like that might be possible in GCC, which supports undoing optimization passes — the compiler could run code generation repeatedly during optimization, and back out the results of any passes that made the generated code fail the verifier. Another audience member noted that they had code that had to run on many possible kernels with different verifiers, and that therefore including any one verifier was insufficient. Dave Thaler indicated that cross-platform BPF compatibility was something that he had a session about later in the day. 

Jowett suggested some less intrusive alternatives, such as permitting more fine-grained control over which passes the optimizer runs, allowing developers to assert that some code compiles in a certain way, or just making the optimizer smarter, before opening the floor for suggestions. Yonghong Song, an LLVM developer, said that he had discussed allowing fine-grained control over the optimizer passes with the upstream project, and with the GCC developers at the 2023 Linux Plumbers Conference. In short, it would be hard. The compilers could add a flag to let the optimizer know it needs to do special verifier-friendly things, but that has not yet been implemented. 

Jowett asked whether Song had any thoughts about code-coverage instrumentation, or whether perhaps BPF programs were too small to benefit. Song thought code coverage was not likely to be extremely useful, but it may still be useful, and invited people interested in the idea to talk with him about it. An audience member suggested that perhaps BPF programs could get code-coverage information already by analyzing the verifier log — which records every instruction it analyzes. Jowett indicated that this was not sufficient, because it does not actually provide any information about whether a particular branch is covered at runtime. Another audience member indicated that they had written a tool that increments counters in a BPF map for each instruction executed, but that the tool had various limitations. 

During the general discussion toward the end of the session, Marchesi asked whether Jowett's idea of preserving likely/unlikely branch hints would require changes to BPF bytecode. Jowett indicated that it would. Another participant noted that they were unsure whether a smarter JIT would be worth it, saying that a more complicated JIT was "really scary" from a memory-model perspective. The BPF JIT runs after the compiler's and verifier's safety checks, so a bug in the JIT is much more likely to break things than an optimization done in the compiler. 

Since the group didn't appear to come to much of a consensus, it seems likely that this will remain a topic of discussion. Modern C compilers are, in some ways, a bad fit for BPF; the verifier cares about many properties of programs that have not been a concern for historical targets. Whether and how the BPF developers will be able to overcome this wrinkle remains to be seen.
