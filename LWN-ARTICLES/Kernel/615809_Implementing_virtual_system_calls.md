---
title: Implementing virtual system calls
url: https://lwn.net/Articles/615809/
date: "October 15, 2014"
category: "System calls-Virtual; vDSO"
author: "October 15, 2014 This article was contributed by Daniel Pierre Bovet"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

October 15, 2014

This article was contributed by Daniel Pierre Bovet

The "virtual dynamic shared object" (or vDSO) is a small shared library exported by the kernel to accelerate the execution of certain system calls that do not necessarily have to run in kernel space. While the kernel developers have settled on a small set of functions to export via the vDSO, there is nothing preventing developers from adding their own. If there is some information an application needs to obtain frequently and quickly from the kernel, a vDSO function might be useful solution. See [the vDSO(7) man page](<http://man7.org/linux/man-pages/man7/vdso.7.html>) for an introduction to the vDSO. 

This article shows how the programming technique used to implement these functions is based mainly on clever additions to the Linux linker script and how the same technique can be applied to implement functions that quickly compute values based on sets of kernel variables. It can be seen as a sort of complement to [this series](<https://lwn.net/Articles/604287/>) on regular system calls. Depending on the hardware platform, different sets of functions are included in the vDSO library. The implementation described here refers to the `x86_64` architecture. 

####  Virtual system calls 

When a process invokes a system call, it executes a special instruction forcing the CPU to switch to kernel mode, saves the contents of the registers on the kernel mode stack, and starts the execution of a kernel function. When the system call has been serviced, the kernel restores the contents of the registers saved on the kernel mode stack and executes another special instruction to resume execution of the user-space process. 

Putting system calls that access kernel-space information into the process address space would make them faster because they would be able to fetch the required value from the kernel address space without those context switches. Clearly, only "read-only" system calls are valid candidates for this type of emulation because user-space processes are not allowed to write into the kernel address space. User-space functions that emulate system calls are called _virtual system calls_. 

The Linux vDSO implementation on `x86_64` offers four of these virtual system calls: `__vdso_clock_gettime()`, `__vdso_gettimeofday()`, `__vdso_time()`, and `__vdso_getcpu()`. They correspond, respectively, to the standard `clock_gettime()`, `gettimeofday()`, `time()`, and `getcpu()` system calls. 

How much faster is a virtual system call than a standard one? This clearly depends on the hardware platform and on the processor type. On a P6T SE ASUS motherboard with an Intel 2.8GHz Core i7 CPU, the average time required to execute a standard `gettimeofday()` system call is 90.5 microseconds, the average time for the corresponding virtual system call is 22.3 microseconds, a significant improvement which justifies the effort spent in developing the vDSO framework. 

####  What is the vDSO?

When thinking about the vDSO, you should keep in mind that this term has two different meanings: (1) it is a dynamic library, but the term is also used to refer to (2) a memory region belonging to the address space of every user-mode process. The vDSO memory region — like most other process memory regions — has its location randomized by default every time it is mapped. Address-space layout randomization is a form of defense against security holes. 

If you display, by means of the "`cat /proc/_pid_ /maps`" command, the memory regions owned by the process having a process ID equal to `_pid_` , you'll get a line like: 

```
7ffffb892000-7ffffb893000 r-xp 00000000 00:00 0          [vdso]
```

which describes the attributes of this special region. The vDSO beginning address, `0x7ffffb892000`, is smaller than `PAGE_OFFSET` (which is `0xffff880000000000` on x86-64 machines), thus, the vDSO is part of the user-space address space. The final address, namely `0x7ffffb893000`, shows that the vDSO occupies a single 4KB page. The `r-xp` permission flags specify that read and execute permissions are enabled and that the region is private (not shared). The last three fields indicate that the region is not mapped from any file and, thus, that it has no inode. 

The binary code stored in the vDSO memory region has the format of a dynamic library. If you dump the code from the vDSO memory region into a file and apply the `file` command to it, you'll get: 

```
ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, stripped
```

All Linux shared dynamic libraries, such as `glibc`, use the ELF format. 

If you disassemble the file containing a vDSO memory region, you'll find the assembly code of the four virtual system calls mentioned previously. On a 3.15 kernel, only 2733 bytes out of 4096 are needed to store the ELF header and the code of the virtual system calls in the vDSO. This means that there is still room for additional functions. 

Since the vDSO is a fully formed ELF image, you can do symbol lookups on it. This allows new symbols to be added with newer kernel releases and allows the C library to detect available functionality at run time when running under different kernel versions. If you run "`readelf -s`" on a file containing a vDSO memory region, you'll get a display of the entries in the symbol table section of the file: 

```
Symbol table '.dynsym' contains 11 entries:
       Num:    Value          Size Type    Bind   Vis      Ndx Name
         0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND 
         1: ffffffffff700330     0 SECTION LOCAL  DEFAULT    7 
         2: ffffffffff700600   727 FUNC    WEAK   DEFAULT   13 clock_gettime@@LINUX_2.6
         3: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  ABS LINUX_2.6
         4: ffffffffff7008e0   365 FUNC    GLOBAL DEFAULT   13 __vdso_gettimeofday@@LINUX_2.6
         5: ffffffffff700a70    61 FUNC    GLOBAL DEFAULT   13 __vdso_getcpu@@LINUX_2.6
         6: ffffffffff7008e0   365 FUNC    WEAK   DEFAULT   13 gettimeofday@@LINUX_2.6
         7: ffffffffff700a50    22 FUNC    WEAK   DEFAULT   13 time@@LINUX_2.6
         8: ffffffffff700a70    61 FUNC    WEAK   DEFAULT   13 getcpu@@LINUX_2.6
         9: ffffffffff700600   727 FUNC    GLOBAL DEFAULT   13 __vdso_clock_gettime@@LINUX_2.6
        10: ffffffffff700a50    22 FUNC    GLOBAL DEFAULT   13 __vdso_time@@LINUX_2.6
```

Here you can see the various functions found within the vDSO region. 

#### Where is the data? 

No mention has been made until now of how virtual system calls retrieve variables from the kernel address space. This is perhaps the most interesting and least documented feature of the vDSO subsystem. Consider, for the sake of concreteness, the `__vdso_gettimeofday()` virtual system call. This function fetches the kernel data it needs from a variable called `vsyscall_gtod_data`. This variable has _two_ different addresses: 

  * The first one is a regular kernel-space address whose value is greater than `PAGE_OFFSET`. If you look at the `System.map` file, you'll find that this symbol has an address like `ffffffff81c76080`. 

  * The second address is in a region called the "`vvar` page". The base address (`VVAR_ADDRESS`) of this page is defined in the kernel to be at `0xffffffffff5ff000`, close to the end of the 64-bit address space. This page is made available read-only to user-space code. 

Clearly, both addresses map to the same physical address; i.e. they refer to the same page frame. 

Variables are created in this page with the `DECLARE_VVAR()` macro. For example, the declaration of `vsyscall_gtod_data` puts it at an offset of 128 within the `vvar` page. The user-space-visible address of this variable is thus: `0xffffffffff5ff000 + 128`. In order to allow the linker to detect the variables exist in the `vvar` page, the `DECLARE_VVAR()` macro puts them in the `.vvar` special section in the kernel binary image (see [Special sections in Linux binaries](<http://lwn.net/Articles/531148/>) for more information). 

Code running within the kernel uses the kernel-space address to access `vsyscall_gtod_data`. Virtual system calls, which run in user mode, must use the second address. The variables located in the vDSO page are accessed by user-mode processes using the `__USER_DS` segment descriptor. They can be read but they cannot be written. As a further precaution, Linux declares them as `const` so that the compiler will detect any attempt to write into them. 

The values of the `vvar` variables are set from the values of other kernel variables not accessible to user-space code. When the kernel modifies the value of one of its internal variables, the associated variable(s) in the `vvar` page must be updated. In Linux, this task is performed by the `timekeeping_update()` function which is invoked, for instance, whenever `jiffies`, the number of elapsed ticks since the system was started, is modified. 

####  Adding a function to the vDSO page 

The Linux vDSO implementation makes it easy for kernel developers to add new functions into the vDSO page. If you look at the code of the four virtual system calls, you'll notice that three of them fetch the kernel data they need from a `vvar` variable called `vsyscall_gtod_data` of type `struct vsyscall_gtod_data`. The fourth one, that is, `__vdso_getcpu()`, does not fetch anything: it gets the CPU index by executing a `rdtscp` instruction. 

Another important observation is that the parameter passed to `timekeeping_update()`, the function which updates the fields of `vsyscall_gtod_data`, is a pointer to a global kernel variable called `timekeeper` of type `struct timekeeper`. When a kernel function updates a field of `timekeeper` related to a virtual system call, it can assume that in a short while this change will affect `vsyscall_gtod_data` and thus the values returned by the virtual system calls. In other words, kernel functions that update fields of `timekeeper` are loosely coupled with virtual system calls. 

The simplest way to define a new vDSO function is to create a similar coupling between the internal kernel variables of interest and a variable added to the `vvar` page. The `update_vsyscall()` function, which you can expect to be called with relatively high frequency, can be augmented to move the data into the `vvar` page as needed. 

Here are some hints that may help you in developing new vDSO functions: 

  * When coding a vDSO function, remember that no external kernel function or global kernel variable can appear in the code, only automatic variables (stack) and `vvar` variables. Since the function runs in user mode, kernel-space identifiers are unknown to the linker. 

  * The Linux linker script that handles vDSO functions must be told that a new function, say `__vdso_foo()`, is being added. To that end, you must add a couple of lines like the following to `linux/arch/x86/vdso/vdso.lds.S`: 

```
foo;
            __vdso_foo;
```

You must also add the following lines at the end of the definition of `__vdso_foo()` in the `linux/arch/x86/vdso/` directory: 

```
int foo(struct fd *fd) __attribute__((weak, alias("__vdso_foo")));
```

In this way, `foo()` becomes a weak alias for `__vdso_foo()`. 

  * Don't forget to modify the code of `update_vsyscall()`, a simple transfer function invoked by `timekeeping_update()`, which copies some fields of `timekeeper` into the `vsyscall_gtod_data` variable. This function can also copy your data of interest into your new variable in the `vvar` page. 

  * If the prototype of the new function is: `int __vdso_foo(struct fd *fd)`, the test program, say `test_foo.c`, which invokes `__vdso_foo()` must be linked as: 

```
gcc -o test_foo linux/arch/x86/vdso/vdso.so test_foo.c
```

because the code of `__vdso_foo()` is included in the `vdso.so` library. Having defined `foo` as a weak alias for `__vdso_foo`, you may also invoke `foo()` instead of `__vdso_foo()` in the test program. 

#### Conclusion

The programming technique used to implement vDSO functions is based on some clever additions to the Linux linker script that allow kernel functions defined in the Linux kernel to be linked in the address space of all user-mode processes. New vDSO functions can be easily implemented to get information about the current kernel state (number of processes in the system, number of free pages, etc.). If you have information that you must get out of the kernel with high frequency and low overhead, the vDSO mechanism might just provide the tool you need.
