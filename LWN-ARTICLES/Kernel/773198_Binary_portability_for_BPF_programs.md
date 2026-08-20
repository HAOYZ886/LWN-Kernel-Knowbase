---
title: Binary portability for BPF programs
url: https://lwn.net/Articles/773198/
date: "November 30, 2018"
category: BPF
author: "By Jonathan Corbet November 30, 2018 Linux
Plumbers Conference"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 30, 2018

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2018-Linux_Plumbers_Conference>)

The BPF virtual machine is the same on all architectures where it is supported; architecture-specific code takes care of translating BPF to something the local processor can understand. So one might be tempted to think that BPF programs would be portable across architectures but, in many cases, that turns out not to be true. During the [BPF microconference](<http://vger.kernel.org/lpc-bpf.html>) at the [Linux Plumbers Conference](<https://linuxplumbersconf.org/>), Alexei Starovoitov (assisted by Yonghong Song, who has done much of the work described) explained the problem and the work that has been done toward "compile once, run everywhere" BPF. 

Many BPF programs are indeed portable, in that they will load and execute properly on any type of processor. Packet-filtering programs, in particular, usually just work. But there is a significant class of exceptions in the form of tracing programs, which are one of the biggest growth areas for BPF. Most tracing tools have two components: a user-space program invoked by the user, and a BPF program that is loaded into the kernel to filter, acquire, and possibly boil down the needed data. Both programs are normally written in C. 

The BPF side of a tracing program may have to dig deeply into the guts of the kernel, and those guts can change significantly from one kernel to the next. The offsets of specific fields within structures are a particular problem; they can differ depending on architecture, kernel configuration options, and more. Tracing programs often need to use those offsets to get the data they are looking for. If the offsets built into a given BPF program do not match the current kernel, the program will not produce the correct results. 

This problem is "solved" now by compiling BPF programs on the fly, just prior to loading them into the kernel. To do that, the [BPF Compiler Collection](<https://github.com/iovisor/bcc>) (BCC) bundles a copy of the Clang compiler, which is a lot of code to haul around — and [![\[Alexei
Starovoitov\]](https://static.lwn.net/images/conf/2018/lpc/AlexeiStarovoitov-sm.jpg)](<https://lwn.net/Articles/773224/>) much of that code has to be linked into the tracing program itself, where it consumes RAM. This toolchain, along with the kernel development headers, must be installed on the system being traced, a painful task on embedded systems. Even then, it's often necessary to paste specific structure definitions into BPF programs to be able to access the needed fields. 

The proposed solution is to introduce structure-field offset information into the [BPF Type Format (BTF) section](<https://lwn.net/Articles/750695/>) describing a compiled BPF program. Those offsets are built into BPF programs by the compiler now; what is needed is a set of pointers to where those offsets are used and their associated field names; then the libbpf library will be enhanced to "relocate" those offsets to match the current kernel before a given program is loaded into the kernel. 

Parts of this problem are hard. In particular, getting the field-name information through LLVM's intermediate representation is difficult; there is "a lot of compiler work" to be done to support this feature. The information needed to perform relocation is more readily available from the `vmlinux` kernel image file on the target system. Ongoing work includes converting the data-type information stored in the DWARF format in the kernel image to BTF, a process that reduces the size of that information from 120MB to 2MB. 

Offsets to structure fields are not the only problem that needs to be solved, though. Imagine a bit of code that looks like: 

```
#if KERNEL_VERSION == 406
            minrtt = ms.v1;
        #else
            minrtt = ms.v2;
        #endif
```

The branch that is pruned by the preprocessor never appears in the output, with the result that the generated BPF code is dependent on the kernel version. The planned solution here is to turn the preprocessor variable into a BPF variable, so that the above code could be written as: 

```
if (__bpf_kernel_version == 406)
            minrtt = ms.v1;
        else
     	minrtt = ms.v2;
```

Both paths are now present in the generated BPF code, which will do the right thing regardless of the kernel version. Other cases are harder; imagine, for example, code that is dependent on whether the `REQ_OP_SHIFT` macro is defined. Once again, a global variable (`__bpf_req_op_shift`) is created to delay the decision until run time and keep all paths present in the generated code. Things get more complicated when it comes to types that may not exist at all depending on something like a configuration variable. Solutions here include a complex "fuzzy struct-type matching" mechanism, or just creating a massive file full of type information (in the BTF format) for a wide range of kernel versions. 

The problem can be made arbitrarily complex, though; Jes Sorensen asked whether it would be possible to handle CPU masks, which are stored on the kernel stack — unless the system is too large, in which case they are pushed out to heap storage. The answer was that some things will just never be possible. 

Other problems include calling static inline functions and preprocessor macros from BPF programs; there does not appear to be a better solution than just copying them into the program at this point. That will bloat the size of the program, of course, and getting some of those functions past the BPF verifier could prove to be a challenge. 

Some related work has to do with adding global variables and read-only data to BPF programs. Globals, which are needed to support some of the techniques described above, can be added without any compiler changes, but the kernel API to support them still needs to be designed and implemented. That is also true of read-only data, which would be especially useful for the handling of strings in BPF programs. 

There are clearly a few things to be worked out in this area still, and it may never be possible to run an arbitrary BPF program on any system. But it seems likely that BPF users will see a solution that works for a lot of the commonly-used tools in the BCC collection, which should make life easier for a lot of use cases. 

(The [slides from this presentation [PDF]](<http://vger.kernel.org/lpc_bpf2018_talks/bpf_compile_once.pdf>) are available.) 

[Thanks to the Linux Foundation, LWN's travel sponsor, for supporting my travel to the event.]
