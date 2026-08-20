---
title: "Reliable user-space stack traces with SFrame"
url: https://lwn.net/Articles/932209/
date: "May 22, 2023"
category: "Development tools-SFrame"
author: "By Jonathan Corbet May 22, 2023 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 22, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

A complete stack trace is needed for a number of debugging and optimization tasks, but getting such traces reliably can be surprisingly challenging. At the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>), Steve Rostedt and Indu Bhagat described a mechanism called SFrame that enables the creation of reliable user-space stack traces in the kernel without the memory and run-time overhead of some other solutions. 

[![\[Steve Rostedt\]](https://static.lwn.net/images/conf/2023/lsfmm/SteveRostedt-sm.png)](<https://lwn.net/Articles/932213/>) Rostedt began by saying that obtaining a full stack trace of a user-space process is useful for a number of purposes. It is needed for accurate profiling, so both perf and ftrace make use of stack traces. BPF programs, too, can benefit from a picture of the state of the call stack. 

The traditional way to reliably obtain stack frames is to build the program in question with frame pointers. The frame pointer is simply a CPU register that is dedicated to containing the base address of the current stack frame. That frame will include a saved copy of the previous frame pointer, indicating where the previous frame began. The kernel (or any other program) can thus follow the chain of frame pointers to locate each frame on the stack. If frame pointers are not present, instead, the kernel's perf subsystem must, at each event, copy a lot of the stack for later postprocessing using the DWARF unwinder. That is a costly thing to do. 

But frame pointers are not free either. Managing the frame pointer requires some setup code to run at the entry to every function. Using a register for the frame pointer makes a scarce CPU register unavailable for other uses, slowing program execution. As described in [this article](<https://lwn.net/Articles/919940/>), building user space with frame pointers can lead to measurable performance regressions, which can cause their use to be controversial. 

The kernel, Rostedt continued, has a stack unwinder called [ORC](<https://lwn.net/Articles/728339/>) that is much simpler than DWARF. It was added in the 4.14 release to support live patching — another application that needs reliable stack traces. The kernel's `objtool` utility creates the ORC data at build time and adds two tables to a section in the kernel executable: `orc_unwind` to hold stack-frame information, and `orc_unwind_ip` to map instruction pointer values to the appropriate unwind entry. 

SFrame is based on ORC; it provides the same mechanism, but for user space rather than the kernel. When an executable is built with SFrame data, the kernel can create full stack traces without the need for frame pointers. There is always a cost, of course; in this case, developers are sacrificing a bit of disk space (to hold the ORC tables) for speed. This data is read, if needed, in the kernel's `ptrace()` path, so it doesn't affect execution when it is not needed. Some additional effort was required to handle some user-space complications; for example, since binaries are relocatable, there must be a mechanism to apply the correct offsets to the SFrame data. 

Rostedt provided an overview of how SFrame support would work in the kernel. The generation of a perf event starts with a non-maskable interrupt (NMI), which ends up in the perf code. If a stack trace is called for, then the kernel will make an attempt to read the call stack; if that encounters a page fault, then there will be no stack trace for this event. He would like to change that code to look for the SFrame data instead. The NMI handler would set a flag indicating that there is work to be done before returning to user space; the `ptrace()` path would see that flag and reconstruct the stack trace in user context. Among other things, that would make it possible to recover the stack even if page faults occur while reading it. 

This approach would require some changes to the user-space perf tool as well. The initial perf event, generated at NMI time, will not include the call stack (which will not be obtained until later), so it will, instead, have a bit set saying "a stack trace is coming". There may be several intervening events generated before that stack trace finally shows up in the ring. Joel Fernandes asked whether the kernel could just reserve space in the ring buffer at NMI time, then fill it in later. Rostedt answered that the ring may end up with multiple events all with the same stack trace; reserving that space for each would end up wasting space. 

[![\[Indu Bhagat\]](https://static.lwn.net/images/conf/2023/lsfmm/InduBhagat-sm.png)](<https://lwn.net/Articles/932214/>) Rostedt concluded his part by saying that the stack is unlikely to be swapped out, so generating the trace will not normally create I/O to fault pages back in. That said, generating the trace will need to bring in some other data, since the SFrame tables are stored in the executable on disk. The SFrame data should only be mapped when it is actually used, so the first use within a process will cause a brief stall while that mapping takes place. 

Bhagat (who has done [much of the work](<https://lwn.net/ml/linux-toolchains/20230501200410.3973453-1-indu.bhagat@oracle.com/>) to implement this functionality) said that there could perhaps be a problem with code in parts of the kernel that are written in assembly. The non-standard stack usage in that code may well confuse the unwinder. It remains to be seen whether unwinding through those parts of the kernel is important, she said. 

Another potential issue is that the SFrame data is stored unaligned on the disk; that can lead to unaligned memory accesses in the kernel. Avoiding that requires a certain amount of copying of data, "weird casts", and such. The alternative, forcing the data to be aligned, would bloat the format though. There seemed to be agreement that storing the data unaligned is the best solution, and that there was no need to change it. 

Other outstanding problems include the need to handle [`dlopen()`](<https://man7.org/linux/man-pages/man3/dlopen.3.html>), which maps executable text from another file into a range of the calling process's memory. This issue could perhaps be addressed by adding a system call to tell the kernel where the SFrame data for a given executable mapping can be found. Just-in-time compiled code is also a problem; when there is no backing file for a mapping, there is no SFrame data either. 

As the session concluded, the sentiment in the room seemed to be that SFrame would be a nice tool to have and that this work should continue.
