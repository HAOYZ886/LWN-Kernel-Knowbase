---
title: "SFrame-based stack unwinding for the kernel"
url: https://lwn.net/Articles/1029189/
date: "July 11, 2025"
category: "Development tools-SFrame; Releases-6.19"
author: "By Jonathan Corbet July 11, 2025"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 11, 2025

The kernel's [perf events subsystem](<https://www.man7.org/linux/man-pages/man2/perf_event_open.2.html>) can produce high-quality profiles, with full function-call chains, of resource usage within the kernel itself. Developers, however, often would like to see profiles of the whole system in one integrated report with, for example, call-stack information that crosses the boundary between the kernel and user space. Support for unwinding user-space call stacks in the perf events subsystem is currently inefficient at best. A long-running effort to provide reliable, user-space call-stack unwinding within the kernel, which will improve that situation considerably, appears to be reaching fruition. 

A process's call stack (normally) contains all of the information that is needed to recreate the chain of function calls that led to a given point in its execution. Each call pushes a frame onto the stack; that frame contains, among other things, the return address for the call. The problem is that exactly _where_ that information lives on the stack is not always clear. Functions can (and do) put other information there, so there may be an arbitrary distance between the address in the stack pointer register and the base of the current call frame at any given time. That makes it hard for the kernel (or any program) to reliably work through the call chain on the stack. 

One solution to the problem is to use frame pointers — dedicating a CPU register to always point to the base of the current call frame. The frame pointer will be saved to the call stack at a known offset for each function call, so frame pointers will reliably make the structure of the call stack clear. Unfortunately, frame pointers hurt performance; they occupy a CPU register, and must be saved and restored on each function call. As a result, building software (in both kernel and user space) without frame pointers is a common practice; indeed, it is the usual case. 

Without frame pointers, the kernel cannot unwind the user-space call stack in the perf events subsystem. The best that can be done is to copy the user-space stack (through the kernel) to the application receiving the performance data, in the hope that the unwinding can be done in user space. Given the resolution at which performance data can be collected, the result is the copying of massive amounts of data, and a corresponding impact on the performance of the system. 

Call-stack unwinding is not a new problem, of course. In user space, the solution has often involved debugging data generated in the [DWARF](<https://dwarfstd.org/>) format. DWARF data is voluminous and complicated to interpret, though; handling it properly requires implementing a special-purpose virtual machine. Trying to use DWARF in the kernel is a good way to lower one's popularity in that community. 

So DWARF is not an answer to this problem. Some years ago, developers working on live patching (which also needs reliable stack unwinding) came up with [a new data format called ORC](<https://lwn.net/Articles/728339/>); it is far simpler than DWARF and relatively compact. ORC has been a part of the kernel's build process ever since, but it has never been adapted to user space. 

#### Enter SFrame

At least, ORC hadn't been adapted to user space until the [SFrame project](<https://lwn.net/Articles/932209/>) (which truly needs an amusing Lord-of-the-Rings-inspired name) was launched. SFrame is an ORC-inspired format that is designed to be as compact as possible. In an executable file that has been built with SFrame data, there is a special ELF section containing two tables that look somewhat like this: 

> ![\[SFrame diagram\]](https://static.lwn.net/images/2025/sframe.svg)

The left-hand table (the "frame descriptor entries") contains one entry for each function; the table is sorted by virtual address. When the need comes to unwind a call stack, the first step is to take the current program counter and, using a binary search, find the frame descriptor entry that includes that address. Each entry contains, beyond the base address for the function, pointers to one or more "frame row entries". Each of those entries contains between one and 15 triplets with start and size fields describing a portion of the function's code, and an associated frame-base offset. The offset of the program counter from the base address is used to find the relevant triplet which, in turn, gives the offset from the current stack pointer to the base of the call frame. From there, the process can be repeated to work up the call stack, one frame at a time. 

The GNU toolchain has the ability to build executables with SFrame data, though it seems that some of that support is still stabilizing. SFrame capability is evidently coming to LLVM soon as well. Thus far, though, the kernel is unable to make use of that data to generate user-space call traces. 

See [this article](<https://lwn.net/Articles/940686/>) for more information about SFrame. There is [a specification for the v2 SFrame format](<https://sourceware.org/binutils/docs/sframe-spec.html>) available; this is the format that the GNU toolchain currently supports. [Version 3 of the SFrame format](<https://sourceware.org/binutils/wiki/sframe/sframev3todo>) is under development, seemingly with the goal of deployment before SFrame is widely used. The new version adds some flexibility for finding the location of the top frame, the elimination of unaligned data fields (which are explicitly used by the current format to reduce its memory requirements), support for signal frames, and s390x architecture support, among other things. 

#### Using SFrame in the kernel

Back in January Josh Poimboeuf posted [a patch series](<https://lwn.net/ml/all/cover.1737511963.git.jpoimboe@kernel.org/>) adding support for SFrame for user-space call-stack unwinding in the perf events subsystem. He has since moved on to other projects; Steve Rostedt has picked up the work and is trying to push it through to completion. The original series is now broken into three core chunks, each of which addresses a part of the problem. 

The SFrame data, being stored in an ELF section, can be mapped into a process's address space along with the rest of the executable. Accessing that data from the kernel, though, is subject to all of the constraints that apply to user-space memory. Notably, this data may be mapped into the address space, but it might not be resident in RAM, so the kernel must be prepared to take page faults when accessing it. 

That, in turn, means that the kernel can only access SFrame data when it is running in process context. Data from the system's performance monitoring unit, though, tends to be delivered via interrupts. So the code that handles those interrupts, and which generates the kernel portion of the stack trace, cannot do the user-space unwinding. That work has to be deferred until a safe time, which is usually just before the kernel returns back to user space. 

So [the first patch series](<https://lwn.net/ml/all/20250708012239.268642741@kernel.org>) adds the infrastructure for this deferred unwinding. When the need to unwind a user-space stack is detected, a work item is set up and attached using the task_work mechanism; the kernel will then ensure that this work is done at a time when user-space memory access is safe. Support for deferred, in-kernel stack unwinding on systems where frame pointers are in use is also added as part of this series. 

The [second series](<https://lwn.net/ml/all/20250708020003.565862284@kernel.org>) teaches the perf events subsystem to use this new infrastructure. The resulting interface for user space is somewhat interesting. A call into the kernel can generate a large number of performance events, each of which may have a stack trace associated with it. But the user-space portion of that trace can only be created just before the return to user space. So the perf events subsystem will report any number of kernel-side traces, followed by a single user-space trace at the end. Since the user-space side of the call chain does not change while this is happening, a single trace is all that is needed. User space must then put the pieces back together to obtain the full trace. The [last patch in the series](<https://lwn.net/ml/all/20250708020051.774149982@kernel.org>) updates the `perf` tool to perform this merging and output unified call chains. 

Finally, the [third series](<https://lwn.net/ml/all/20250708021115.894007410@kernel.org>) adds SFrame support to all of this machinery. When an executable is loaded, any SFrame sections are mapped as well. Within the kernel, a [maple tree](<https://lwn.net/Articles/845507/>) is used to track the SFrame sections associated with each text range in the executable. The kernel does not, however, track the SFrame sections associated with shared libraries, so the series contains a new [`prctl()`](<https://man7.org/linux/man-pages/man2/prctl.2.html>) operation so that the C library can provide that information explicitly. That part of the API is likely to change (the [relevant patch](<https://lwn.net/ml/all/20250708021200.397301537@kernel.org>) suggests that a separate system call should be added) before the series is merged. 

With the SFrame information at hand, using it to generate stack traces is a relatively straightforward job; the kernel just has to use the SFrame tables to work through the frames on the stack. The result should be fast and reliable, with a minimal impact on the performance of the workload being measured. 

The sum total of this work is quite a bit of new infrastructure added to the core kernel, including code that runs in some of the most constrained contexts. So the chances are good that there are still some minor problems to work out. The bulk of the hard work would appear to be done, though. It may take another development cycle or three, but the multi-year project to get SFrame support into the kernel would appear to be reaching its conclusion.
