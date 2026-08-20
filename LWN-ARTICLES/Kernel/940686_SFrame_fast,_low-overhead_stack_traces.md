---
title: "SFrame: fast, low-overhead stack traces"
url: https://lwn.net/Articles/940686/
date: "August 8, 2023"
category: "Development tools-SFrame"
author: "By Jake Edge August 8, 2023 OSSNA"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
August 8, 2023

* * *

[OSSNA](<https://lwn.net/Archives/ConferenceByYear/#2023-Open_Source_Summit_North_America>)

Getting a stack trace of a running program is useful in a variety of scenarios: tracing, profiling, debugging, performance tuning, and more. There are existing mechanisms to get stack traces, but there are some downsides to them; the "Simple Frame" (SFrame) stack-trace format came about to address the shortcomings in the other techniques. Back in May, Steve Rostedt and Indu Bhagat gave a [talk about SFrame support in the kernel](<https://lwn.net/Articles/932209/>) as part of [LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023/>); a few days later, Bhagat gave a more general talk about SFrame ([YouTube video](<https://www.youtube.com/watch?v=4XrFYpjyodo>)) at [Open Source Summit North America](<https://events.linuxfoundation.org/open-source-summit-north-america/>) in Vancouver. That second talk helped fill in some other aspects of SFrame and the overall stack-tracing picture. 

#### Background

Bhagat began by defining what a stack trace is: "a list of function calls that are currently active in the thread". A useful stack trace will show information about the instruction pointer (IP) location within each function in the call-chain list, as well as some human-readable symbol names. That includes the function name, but it can also provide information like file names and line numbers. The symbolization piece is beyond the scope of her talk, however, which focuses on how to get the list of IPs in the call chain. 

[ ![\[Indu Bhagat\]](https://static.lwn.net/images/2023/ossna-bhagat-sm.png) ](<https://lwn.net/Articles/940782/>)

Different kinds of tools will generate the call-chain IPs in different ways because they are focused on their own use cases; "a debugger will do it differently than a profiler". From her [slides](<https://static.sched.com/hosted_files/ossna2023/d1/OSS_LinuxCon_SFrame_final.pdf?_gl=1>), she put up a list of five existing stack-tracing mechanisms, but said that she would be talking about three in particular: frame pointer, EH frame, and application-specific formats. Those three will provide enough context to explain the motivations behind the SFrame format, she said. 

The frame-pointer technique is perhaps one of the oldest stack-tracing methods. It reserves a register to hold the frame pointer, which is a pointer to the start of the current stack frame; the compiler generates some extra code to save and restore the value of the stack pointer to that register on function entry and exit. So there is some performance impact of the extra code for each function call; beyond that, the compiler has to reserve the register just for the frame pointer, which also has performance implications. But it is an easy-to-understand mechanism that works well; "it's beautiful, it just works and it is so simple". 

The EH-frame mechanism is a [DWARF](<https://dwarfstd.org/>)-based method that can do more than just stack tracing; it can also do stack unwinding, which means that it can restore the state of all of the registers at each point in the call chain. The information needed to do that is stored in the binaries in `.eh_frame` and `.eh_frame_hdr` ELF sections. The format itself is compact, versatile, and works well in practice, Bhagat said; it also has good library support for applications that want to handle the format. 

Using EH frame does not require reserving a register for the frame pointer but "the stack tracer itself is slow and it is complex". The reason is that the DWARF information is a set of instructions needed to recover the stack offsets for the information of interest; some of the instructions are simple and some complex, "but you would need to implement sort of a stack machine where you can execute the opcodes". The main complaints about this method are its speed and complexity, which also make it unsuitable for use in the kernel. She noted that the [discussion around](<https://lwn.net/ml/fedora-devel/CA+voJeXhBPph9R2nzV9UCiMOU4Tpo0s0NtA26HcJccTwFad25w@mail.gmail.com/>) the now-accepted [Fedora 37 proposal](<https://fedoraproject.org/wiki/Changes/fno-omit-frame-pointer>) to enable frame pointers by default in the builds for the distribution (which was [covered by LWN](<https://lwn.net/Articles/919940/>)) also touched on some of the problems with the frame pointer and EH-frame methods. 

The application-specific formats have come about because of these shortcomings. For example, the kernel's [ORC](<https://lwn.net/Articles/728339/>)-based stack tracing is used because of the problems with the EH-frame method; there are other application-specific formats, which are not open source, but also demonstrate the need for a fast and simple stack-tracing solution. The application-specific solutions are not using information generated by the toolchain, so they may require reverse-engineering to use the formats in other ways; that may make it difficult to port and maintain those formats. 

#### Requirements

She put up a slide that summarized the pros and cons of the three methods, which could be used to generate a set of requirements for a new stack-tracing method. The first was that given any program counter (or instruction pointer, she used both in the talk), a precise stack trace can be generated from it, which she called "asynchronous stack tracing". At the end of the talk, an audience member asked about what that term means in this context. Bhagat said that frame-pointer-based stack traces are not always precise because the compiler adds extra instructions in the prologue and epilogue of a function. If the stack trace starts on one of those instructions, some part of the trace will be missed because the frame-pointer handling is incomplete at those points. 

The other requirements were more obviously derived from the pros and cons on her slide: low-overhead tracing, with a low-complexity tracer, using information that is generated by the toolchain. SFrame has been designed with those requirements in mind, she said. 

The SFrame format was defined and implemented in [Binutils 2.40](<https://sourceware.org/pipermail/binutils/2023-January/125671.html>) as SFrame version 1\. Since the talk, [Binutils 2.41 was released](<https://sourceware.org/pipermail/binutils/2023-July/128719.html>) with some fairly small, but not backward compatible, changes to the format, which is now [SFrame version 2](<https://sourceware.org/binutils/docs/sframe-spec.html>). The format simply contains enough information to be able to do a stack trace: given a program counter (PC) value, the canonical frame address (CFA), frame pointer (FP), and return address (RA) can be retrieved. "That's all you need to stack trace and that is all that is encoded in the Simple Frame stack-trace format." 

SFrame is defined for two ABIs: x86_64 and 64-bit Arm. It has support for encoding [procedure linkage table](<https://refspecs.linuxfoundation.org/ELF/zSeries/lzsabi0_zSeries/x2251.html#PROCEDURELINKAGETABLE>) entries (pltN). On Arm, it can encode [pointer authentication](<https://lwn.net/Articles/718888/>) information so that the return address values that have been mangled by the authentication can be decoded. 

The SFrame information is stored in a `.sframe` ELF section, which is stored in its own `PT_GNU_SFRAME` segment. To generate it, the `‑‑gsframe` option needs to be passed to the GNU assembler. The GNU linker "will do the right thing" if it sees multiple `.sframe` sections by combining them in the output. The `readelf` and `objdump` tools also have support for SFrame; using the `‑‑sframe` option will provide human-readable text describing the SFrame information. 

#### Format

The SFrame format has three parts: a header, a set of function descriptor entities (FDEs), and a set of frame row entries (FREs). The header contains a magic number, a version number, and offsets to the other two sections. The FDEs are fixed size, sorted in PC order, so a binary search can be used to find the function corresponding to a given PC value. FREs are variable-length in order to be as compact as possible. Offsets are used to access the various pieces of information in the format. 

Each FDE corresponds to a single function. It stores the start PC and function size in bytes; it also indicates whether it is a regular code block or a pltN. After that, it has an offset to the first FRE along with the number and types of FREs that the function has. 

The FREs are the backbone of the format, she said. They provide the stack offsets that can be used to recover the CFA, FP, and RA given a particular PC within the function. Because function sizes differ, the space needed to represent an offset from the starting PC value also differs; there are three different representations for FREs depending on whether the offsets can be encoded in one, two, or four bytes. Each FRE covers a contiguous range of addresses within the function and encodes the stack offsets for the CFA, FP, and RA values that are valid for that range. 

The ability to perform a binary search on the FDEs is one advantage that SFrame has; it allows speedy access to the starting point for a trace. Another advantage for the format is that the FREs directly encode the offsets needed to recover the CFA, FP, and RA; there is no need to execute stack-machine instructions to do so. The kernel's ORC format also directly encodes these offsets, but SFrame does some space optimizations to make its format more compact. She put up a graph ("take it with some grain of salt") showing the space savings on x86_64 for SFrame versus EH frame for ten different binaries from Binutils; it showed that SFrame was roughly 80% of the size required for EH frame. She did caution that the use case for EH frame is different, so it is not exactly a fair comparison. 

#### Library

The libsframe format library ships with Binutils (starting with 2.40); it has APIs to read and write SFrame data; the library was created with the linker in mind, thus it includes a write API that a stack tracer likely will not need. There are also APIs that target stack tracers, such as for finding the FRE that corresponds to a PC value or getting the stack offset from an FRE. 

Libsframe is a young library, so it is too early to make any ABI stability guarantees, Bhagat said. The API is described in `sframe-api.h`. The SFrame format is not aligned on disk, but the library internally arranges the data so that there are no unaligned accesses. She showed some sample code to demonstrate "how easy it is to stack walk"; it found an FRE based on a PC value (`find_fre()`), then got the offsets for the CFA, FP, and RA values (`get_*_offset()`) in order to retrieve them. 

There is some future work needed in the assembler to support a [CFI directive](<https://sourceware.org/binutils/docs/as/CFI-directives.html>) (`.cfi_escape`) that has been skipped at this point; it means that SFrame is not fully asynchronous, but the compiler rarely emits that directive so it is not a large problem in practice. There is also a need to add more regression tests for SFrame unwinding to the tests for the GNU assembler. Beyond that, she said that the SFrame developers plan to work with the community on use cases for SFrame, including for user-space applications and [user-space stack tracing for the kernel](<https://lwn.net/ml/linux-toolchains/20230501200410.3973453-1-indu.bhagat@oracle.com/>). Adding [SFrame support to LLVM](<https://github.com/llvm/llvm-project/issues/64449>) has been suggested since the talk, as well. 

Bhagat finished her talk by suggesting that those who are interested in working with SFrame get in touch with the developers via the [Binutils mailing list](<https://sourceware.org/mailman/listinfo/binutils>). An audience member asked about applications currently using SFrame; Bhagat said that other than the work on the kernel piece, which could be used by perf, Ftrace, BPF, and others, there are no applications using the format. The SFrame developers started with the kernel use case and are now starting to look into user-space applications that could benefit from fast, low-overhead stack traces. 

An attendee asked about other architectures, noting that he believed the RISC-V ABI was rather different. Bhagat said that SFrame already accommodated the differences between x86_64 and Arm64, but if another architecture had major differences in the way it handled its return address, SFrame would probably need to be changed to handle it. As it stands, x86_64 always uses the stack for its RA, while Arm64 uses both the stack and a dedicated register, which SFrame already handles. 

Bhagat's colleague, Jose Marchesi, asked about the relationship between SFrame and ORC; he wondered why the kernel would need something like SFrame instead of simply using ORC. Bhagat said that because ORC is application-specific, it can represent the stack usage of all of the different kinds of code in the kernel, including its hand-rolled assembly code. But to do user-space stack tracing, the ORC format will need some changes; SFrame is not meant to replace the kernel's internal use of ORC, which has similar goals to those of SFrame, but to complement it for doing user-space tracing from the kernel.
