---
title: BPF as a safer kernel programming environment
url: https://lwn.net/Articles/909095/
date: "September 23, 2022"
category: BPF
author: "September 23, 2022 This article was contributed by David Vernet LPC"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

September 23, 2022

This article was contributed by David Vernet

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2022-Linux_Plumbers_Conference>)

For better or worse, C is the _lingua franca_ in the world of kernel engineering. The core logic of the Linux kernel is written entirely in C (with a bit of assembly), as are its drivers and modules. While C is rightfully celebrated for its powerful yet simple semantics, it is an older language that lacks many of the features present in modern languages such as [Rust](<https://lwn.net/Articles/907685/>). The [BPF](<https://lwn.net/Articles/599755/>) subsystem, on the other hand, provides a programming environment that allows engineers to write programs that can run safely in kernel space. At the [2022 Linux Plumbers Conference](<https://lpc.events>) in Dublin, Ireland, Alexei Starovoitov presented an overview of how BPF has evolved over the years to provide a new model for kernel programming. 

#### The mission of BPF

Starovoitov began by describing his "mission statement" for BPF: "To innovate and enable others to innovate". Programming in the kernel has historically taken place in one of two contexts: 

  * _Core kernel programming_ , which includes major core subsystems such as the memory manager, the scheduler, read-copy-update, and more. 
  * _Kernel-module programming_ , which refers to building objects that are not compiled into the main kernel image, and which are, instead, loaded by the module loader at a later time. For example, drivers are written as kernel modules, as are other features, such as filesystems, network protocols, and more. 

This was the state of the kernel for a long time, until the initial extended BPF (eBPF) virtual machine was added to the kernel in version 3.15. With this, BPF programs could be written in a highly restrictive version of C that was compiled into BPF bytecode and which would allow users to write code that is verifiably safe to run in kernel space. [![\[Alexei
Starovoitov\]](https://static.lwn.net/images/conf/2022/lpc/AlexeiStarovoitov-sm.png)](<https://lwn.net/Articles/907943/>)

Since then, BPF has steadily grown both in terms of the size of the code and in the size of the community of users and contributors. According to Starovoitov, email traffic reaches 50-70 messages being received every day on the BPF mailing list and approximately 2000 emails being received per month. The number of active monthly contributors to BPF has grown in tandem as well, reaching approximately 140 as of September 2022. At this point, a majority of the contributions to the BPF subsystem come from outside the Meta BPF group. 

#### The BPF programming environment

While most BPF programs are written in C and compiled with the LLVM Clang compiler, BPF programs are just binary BPF bytecode object files, and do not need to be written in a particular language. For example, BPF programs can be written in Rust using [Aya](<https://lwn.net/Articles/859784/>), or even directly in BPF assembly language. That said, C is the canonical programming language for BPF programs; Starovoitov’s presentation continued with an overview of how the C programming environment has evolved for BPF programs. 

This new programming environment is implemented with a combination of C language extensions and a runtime environment featuring collaboration between Clang, the user-space BPF loader library (libbpf), and the BPF subsystem in the kernel. To create a BPF program, the user writes a program in a C language which is emitted as [BPF instructions](<https://docs.kernel.org/bpf/instruction-set.html>) by a Clang backend implementation. In order to run a program, libbpf loads the BPF program into memory, performs relocations on the program to make it portable across platforms and different kernel versions, and then calls into the kernel to load the program. Finally, in the kernel, the verifier statically verifies that the program is safe to run, and then enables it. 

The BPF programming environment was not always so rich, however. In the early days of BPF, programs were required to use what Starovoitov called "restricted C". All functions in a BPF program had to be fully inlined, loops, static and global variables, and memory allocations were all disallowed. There was also no type information, so BPF programs could only receive a single, fixed input context for tracing and network-filtering functions. 

While it was useful to write BPF programs even in such a highly restrictive environment, it was clear that there was significant opportunity to extend the use cases supported by BPF. One such extension was allowing static functions in BPF programs. Doing so required using libbpf to perform relocations in kernel BPF programs at program load time. Support for [bounded loops](<https://lwn.net/Articles/794934/>) was eventually added after years of designs and attempts, as were iterators. 

#### Extending the programming environment past full C

While this brought BPF closer to full C support, it eventually became clear that BPF programs required features that were not available even in the full C language standard. It was at this point that the BPF community began to extend the BPF programming environment to include new features that distinguished it from traditional C. One of those extensions is Compile Once - Run Everywhere (CO-RE). 

CO-RE makes BPF programs portable across different kernel versions and platforms. It is common in BPF programs to access kernel data structures. The kernel provides no ABI guarantees for struct layouts, however, so a BPF program doing a read at a static offset into a kernel structure could read the wrong value if that structure changes in a future version or a different configuration of the kernel. CO-RE addresses this by leveraging the [BPF Type Format (BTF)](<https://docs.kernel.org/bpf/btf.html>) data present in the running kernel. When a program is loaded, libbpf performs relocations for all struct accesses so that the fields being accessed match the offsets of the fields according to the BTF information of the currently running kernel. 

Starovoitov described a number of other interesting extensions to the BPF programming environment as well. One such feature is [kptrs](<https://lwn.net/Articles/900749/>), which allows pointers to kernel memory to be stored in BPF maps. Another is allowing programs to access kernel-configuration parameters at load time. Kernel modules can only use the configuration values that were set when they were compiled, but BPF programs can adjust to the current kernel's configuration when they are loaded. Yet another feature is "type tags", which allow programs to annotate variables to describe how they’re meant to be used. For example, kptrs can be annotated with `__kptr` and `__kptr_ref` type tags to show that they’re either unreferenced or referenced kptrs respectively. Eventually, pointers may similarly be annotated with `__user` or `__percpu` to tell the compiler and the verifier that they point to user memory or per-CPU memory respectively. 

#### Plans for the future

More extensions are currently being designed and implemented as well, including lock-correctness verification and allowing BPF programs to include assertions. Lock verification would seem at first glance to be a difficult problem to solve, though Dave Marchevsky and Kumar Kartikeya Dwivedi have both already sent out RFC patch sets for new map types with verified locking. [Marchevsky’s patch set](<https://lwn.net/ml/bpf/20220722183438.3319790-1-davemarchevsky@fb.com/>) proposes a new red-black-tree map type, whereas [Dwivedi’s patch set](<https://lwn.net/ml/bpf/20220904204145.3089-1-memxor@gmail.com/>) proposes a list map type. Both patch sets implement semantics that allow BPF programs to perform locking which is checked and validated by the verifier. 

Assertion verification is still in the planning phase, and will potentially be complex to implement. Assertions will serve as a signal to both the compiler and the verifier, with assertions being used to indicate some invariant in the program whose failure should cause the program to abort. Starovoitov claimed that figuring out how to implement program abort would be a "fun" problem, as it requires safe stack unwinding, invoking kptr destructors, and possibly more. 

Starovoitov concluded his presentation by sharing his vision for the future of BPF: replacing kernel modules as the de-facto means of extending the kernel. Whereas the early versions of BPF programs looked more like user-space programs with fixed sets of BPF helper functions and fixed map types, the new BPF allows users to extend the kernel in ways that fit more individualized use cases. Such use cases have in fact already been proposed in the upstream community. Benjamin Tissoires, who spoke at LPC following Starovoitov, has been iterating on a [patch set](<https://lwn.net/ml/bpf/20220902132938.2409206-1-benjamin.tissoires@redhat.com/>) that allows human-input device (HID) quirks to be fixed with BPF programs. No kernel module has fully been replaced by a BPF program as of yet, though it will be interesting to see what other parts of the kernel can be implemented in BPF programs moving forward. 

An audience member asked for more details on the lock-correctness verification that Starovoitov had alluded to. Starovoitov said it was still a work in progress, but that he was optimistic that a way to do static lock checking that verifies proper data protection and guarantees that no deadlocks can occur could be found. Dave Miller responded that, if locks could be statically checked by the verifier, it may be worth investigating whether the locking logic could be automatically _generated_ by the verifier. Starovoitov responded that this was what they were hoping to achieve, with the current design aggregating locks and the data under protection as part of the same allocation. For data that cannot be aggregated with a lock, a BTF Type tag could be used to specify that it needs explicit lock protection.
