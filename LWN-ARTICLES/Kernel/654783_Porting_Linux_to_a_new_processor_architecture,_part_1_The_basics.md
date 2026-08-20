---
title: "Porting Linux to a new processor architecture, part 1: The basics"
url: https://lwn.net/Articles/654783/
date: "August 26, 2015"
category: "Architectures-Porting to"
author: "August 26, 2015 This article was contributed by Joël Porquet"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

August 26, 2015

This article was contributed by Joël Porquet

Although a simple port may count as little as 4000 lines of code—exactly 3,775 for the mmu-less Hitachi 8/300 recently reintroduced in Linux 4.2-rc1—getting the Linux kernel running on a new processor architecture is a difficult process. Worse still, there is not much documentation available describing the porting process. The aim of this series of three articles is to provide an overview of the procedure, or at least one possible procedure, that can be followed when porting the Linux kernel to a new processor architecture.

After spending countless hours becoming almost fluent in many of the supported architectures, I discovered that a well-defined skeleton shared by the majority of ports exists. Such a skeleton can logically be split into two parts that intersect a great deal. The first part is the boot code, meaning the architecture-specific code that is executed from the moment the kernel takes over from the bootloader until `init` is finally executed. The second part concerns the architecture-specific code that is regularly executed once the booting phase has been completed and the kernel is running normally. This second part includes starting new threads, dealing with hardware interrupts or software exceptions, copying data from/to user applications, serving system calls, and so on.

#### Is a new port necessary?

As LWN [reported](<https://lwn.net/Articles/597351/>) about another porting experience in an article published last year, there are three meanings to the word "porting".

It can be a port to a new board with an already-supported processor on it. Or it can be a new processor from an existing, supported processor family. The third alternative is to port to a completely new architecture. 

Sometimes, the answer to whether one should start a new port from scratch is crystal clear—if the new processor comes with a new instruction set architecture (ISA), that is usually a good indicator. Sometimes it is less clear. In my case, it took me a couple weeks to figure out this first question.

At the time, May 2013, I had just been hired by the French academic computer lab [LIP6](<http://www.lip6.fr/?LANG=en>) to port the Linux kernel to [TSAR](<https://www-soc.lip6.fr/trac/tsar>), an academic processor architecture that the system-on-chip research group was designing. TSAR is an architecture that follows many of the current trends: lots of small, single-issue, energy-efficient processor cores around a scalable network-on-chip. It also adds some nice innovations: a full-hardware cache-coherency protocol for both data/instruction caches and translation lookaside buffers (TLBs) as well as physically distributed but logically shared memory.

My dilemma was that the processor cores were compatible with the MIPS32 ISA, which meant the port could fall into the second category: "new processor from an existing processor family". But since TSAR had a virtual-memory model radically different from those of any MIPS processors, I would have been forced to drastically modify the entire MIPS branch in order to introduce this new processor, sometimes having almost no choice but to surround entire files with `#ifndef TSAR ... #endif`.

Quickly enough, it came down to the most logical—and interesting—conclusion: 

```
mkdir linux/arch/tsar
```

#### Get to know your hardware

_Really_ knowing the underlying hardware is definitely the fundamental, and perhaps most obvious, prerequisite to porting Linux to it.

The specifications of a processor are often—logically or physically—split into a least two parts (as were, for example, the recently published specifications for the new [RISC-V](<http://www.riscv.org/>) processor). The first part usually details the user-level ISA, which basically means the list of user-level instructions that the processor is able to understand—and execute. The second part describes the privileged architecture, which includes the list of kernel-level-only instructions and the various system registers that control the processor status.

This second part contains the majority—if not the entirety—of the information that makes a port special and thus often prevents the developer from opportunely reusing code from other architectures.

Among the important questions that should be answered by such specifications are:

  * What are the virtual-memory model of the processor architecture, the format of the page table, and the translation mechanism?

Many processor architectures (e.g. x86, ARM, or TSAR) define a flexible virtual-memory layout. Their virtual address space can theoretically be split any way between the user and kernel spaces—although the default layout for 32-bit processors in Linux usually allocates the lower 3GiB to user space and reserves the upper 1GiB for kernel space. In some other architectures, this layout is strongly constrained by the hardware design. For instance, on MIPS32, the virtual address space is statically split into two regions of the same size: the lower 2GiB is dedicated to user space and the upper 2GiB to kernel space; the latter even contains predefined windows into the physical address space.

The format of the page table is intimately linked to the translation mechanism used by the processor. In the case of a hardware-managed mechanism, when the TLB—a hardware cache of limited size containing recently used translations between virtual and physical addresses—does not contain the translation for a given virtual address (referred to as _TLB miss_), a hardware state machine will transparently fetch the proper translation from the page table structure in memory and fill the TLB with it. This means that the format of the page table must be fixed—and certainly defined by the processor's specifications. In a software-based mechanism, a TLB miss exception is handled by a piece of code, which theoretically leaves complete liberty as to how the page table is organized—only the format of TLB entries is specified.

  * How to enable/disable the interrupts, switch from privileged mode to user mode and vice-versa, get the cause of an exception, etc.?

Although all these operations generally only involve reading and/or modifying certain bit fields in the set of available system registers, they are always very particular to each architecture. It is for this reason that, most of the time, they are actually performed by small chunks of dedicated assembly code.

  * What is the ABI?

Although one could think that the Application Binary Interface (ABI) is only supposed to concern compilation tools, as it defines the way the stack is formatted into stack-frames, the ways arguments and return values are given or returned by functions, etc.; it is actually absolutely necessary to be familiar with it when porting Linux. For example, as the recipient of system calls (which are typically defined by the ABI), the kernel has to know where to get the arguments and how to return a value; or on a context switch, the kernel must know what to save and restore, as well as what constitutes the context of a thread, and so on.

#### Get to know the kernel

Learning a few kernel concepts, especially concerning the memory layout used by Linux, will definitely help. I admit it took me a while to understand what exactly was the distinction between _low memory_ and _high memory_ , and between the _direct mapping_ and _vmalloc_ regions.

For a typical and simple port (to a 32-bit processor), in which the kernel occupies the upper 1GiB of the virtual address space, it is usually fairly straightforward. Within this 1GiB, Linux defines that the lower portion of it will be directly mapped to the lower portion of the system memory (hence referred to as low memory): meaning that if the kernel accesses the address `0xC0000000`, it will be redirected to the physical address `0x00000000`.

In contrast, in systems with more physical memory than that which is mappable in the direct mapping region, the upper portion of the system memory (referred to as high memory) is not normally accessible to the kernel. Other mechanisms must be used, such as `kmap()` and `kmap_atomic()`, in order to gain temporary access to these high-memory pages.

Above the direct mapping region is the vmalloc region that is controlled by `vmalloc()`. This allocation mechanism provides the ability to allocate pages of memory in a virtually contiguous way in spite of the fact that these pages may not necessarily be physically contiguous. It is particularly useful for allocating a large amount of memory pages in a virtually contiguous manner, as otherwise it can be impossible to find the equivalent amount of contiguous free physical pages.

Further reading about the memory management in Linux can be found in [_Linux Device Drivers_ [PDF]](</images/pdf/LDD3/ch15.pdf>) and this [LWN article](<https://lwn.net/Articles/356378/>).

#### How to start?

With your head full of the processor's specifications and kernel principles, it is finally time to add some files to this newly created arch directory. But wait ... where and how should we start? As with any porting or even any code that must respect a certain API, the procedure is a two-step process.

First, a minimal set of files that define a minimal set of symbols (functions, variables, defines) is necessary for the kernel to even compile. This set of files and symbols can often be deduced from compilation failures: if compilation fails because of a missing file/symbol, it is a good indicator that it should probably be implemented (or sometimes that some configuration options should be modified). In the case of porting Linux, this approach is particularly relevant when implementing the numerous headers that define the API between the architecture-specific code and the rest of the kernel.

After the kernel finally compiles and is able to be executed on the target hardware, it is useful to know that the boot code is very sequential. That allows many functions to stay empty at first and to only be implemented gradually until the system finally becomes stable and reaches the `init` process. This approach is generally possible for almost all of the C functions executed after the early assembly boot code. However it is advised to have the `early_printk()` infrastructure up and working otherwise it can be difficult to debug.

#### Finally getting started: the minimal set of non-code files

Porting the compilation tools to the new processor architecture is a prerequisite to porting the Linux kernel, but here we'll assume it has already been performed. All that is left to do in terms of compilation tools is to build a cross-compiler. Since at this point it is likely that porting a standard C library has not been completed (or even started), only a stage-1 cross-compiler can be created.

Such a cross-compiler is only able to compile source code for bare metal execution, which is a perfect fit for the kernel since it does not depend on any external library. In contrast, a stage-2 cross-compiler has built-in support for a standard C library.

The first step of porting Linux to a new processor is the creation of a new directory inside `arch/`, which is located at the root of the kernel tree (e.g. `linux/arch/tsar/` in my case). Inside this new directory, the layout is quite standardized:

  * `configs/`: default configurations for supported systems (i.e. `*_defconfig` files)
  * `include/asm/` for the headers dedicated to internal use only, i.e. Linux source code
  * `include/uapi/asm` for the headers that are meant to be exported to user space (e.g. the libc)
  * `kernel/`: general kernel management
  * `lib/`: optimized utility routines (e.g. `memcpy()`, `memset()`, etc.)
  * `mm/`: memory management

The great thing is that once the new arch directory exists, Linux automatically knows about it. It only complains about not finding a Makefile, not about this new architecture:

```
~/linux $ make ARCH=tsar
        Makefile: ~/linux/arch/tsar/Makefile: No such file or directory
```

As shown in the following example, a minimal arch Makefile only has a few variables to specify:

```
KBUILD_DEFCONFIG := tsar_defconfig
    
        KBUILD_CFLAGS += -pipe -D__linux__ -G 0 -msoft-float
        KBUILD_AFLAGS += $(KBUILD_CFLAGS)
    
        head-y := arch/tsar/kernel/head.o
    
        core-y += arch/tsar/kernel/
        core-y += arch/tsar/mm/
    
        LIBGCC := $(shell $(CC) $(KBUILD_CFLAGS) -print-libgcc-file-name)
        libs-y += $(LIBGCC)
        libs-y += arch/tsar/lib/
    
        drivers-y += arch/tsar/drivers/
```

  * `KBUILD_DEFCONFIG` must hold the name of a valid default configuration, which is one of the `defconfig` files in the `configs` directory (e.g. `configs/tsar_defconfig`).
  * `KBUILD_CFLAGS` and `KBUILD_AFLAGS` define compilation flags, respectively for the compiler and the assembler.
  * `{head,core,libs,...}-y` list the objects (or subdirectory containing the objects) to be compiled in the kernel image (see [Documentation/kbuild/makefiles.txt](<https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt>) for detailed information)

Another file that has its place at the root of the arch directory is `Kconfig`. This file mainly serves two purposes: it defines new arch-specific configuration options that describe the features of the architecture, and it selects arch-independent configuration options (i.e. options that are already defined elsewhere in Linux source code) that apply to the architecture.

As this will be the main configuration file for the newly created arch, its content also determines the layout of the menuconfig command (e.g. `make ARCH=tsar menuconfig`). It is difficult to give a snippet of the file as it depends very much on the targeted architecture, but looking at the same file for other (simple) architectures should definitely help.

The `defconfig` file (e.g. `configs/tsar_defconfig`) is necessary to complete the files related to the Linux kernel build system (kbuild). Its role is to define the default configuration for the architecture, which basically means specifying a set of configuration options that will be used as a seed to generate a full configuration for the Linux kernel compilation. Once again, starting from defconfig files of other architectures should help, but it is still advised to refine them, as they tend to activate many more features than a minimalistic system would ever need—support for USB, IOMMU, or even filesystems is, for example, too early at this stage of porting.

Finally the last "not really code but still really important" file to create is a script (usually located at `kernel/vmlinux.lds.S`) that will instruct the linker how to place the various sections of code and data in the final kernel image. For example, it is usually necessary for the early assembly boot code to be set at the very beginning of the binary, and it is this script that allows us do so.

#### Conclusion

At this point, the build system is ready to be used: it is now possible to generate an initial kernel configuration, customize it, and even start compiling from it. However, the compilation stops very quickly since the port still does not contain any code.

In the next article, we will dive into some code for the second portion of the port: the headers, the early assembly boot code, and all the most important arch functions that are executed until the first kernel thread is created.
