---
title: Special sections in Linux binaries
url: https://lwn.net/Articles/531148/
date: "January 3, 2013"
category: Build system
author: "January 3, 2013 This article was contributed by Daniel Pierre Bovet"
---

January 3, 2013

This article was contributed by Daniel Pierre Bovet

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

A section is an area in an object file that contains information which is useful for linking: the program's code and data, relocation information, and more. It turns out that the Linux kernel has some additional types of sections, called "special sections", that are used to implement various kernel features. Special sections aren't well known, so it is worth shedding some light on the topic. 

####  Segments and sections 

Although Linux supports several binary file formats, ELF ([Executable and Linking Format](<http://en.wikipedia.org/wiki/Executable_and_Linkable_Format>)) is the preferred format since it is flexible and extensible by design, and it is not bound to any particular processor or architecture. ELF binary files consist of an ELF header followed by a few segments. Each segment, in turn, includes one or more sections. The length of each segment and of each section is specified in the ELF header. Most segments, and thus most sections, have an initial address which is also specified in the ELF header. In addition, each segment has its own access rights. 

The linker merges together all sections of the same type included in the input object files into a single section and assigns an initial address to it. For instance, the `.text` sections of all object files are merged together into a single `.text` section, which by default contains all of the code in the program. Some of the segments defined in an ELF binary file are used by the GNU loader to assign memory regions with specific access rights to the process. 

Executable files include four canonical sections called, by convention, `.text`, `.data`, `.rodata`, and `.bss`. The `.text` section contains executable code and is packed into a segment which has the read and execute access rights. The `.data` and `.bss` sections contain initialized and uninitialized data respectively, and are packed into a segment which has the read and write access rights. 

Linux loads the `.text` section into memory only once, no matter how many times an application is loaded. This reduces memory usage and launch time and is safe because the code doesn't change. For that reason, the `.rodata` section, which contains read-only initialized data, is packed into the same segment that contains the `.text` section. The `.data` section contains information that could be changed during application execution, so this section must be copied for every instance. 

The "`readelf -S`" command lists the sections included in an executable file, while the "`readelf -l`" command lists the segments included in an executable file. 

####  Defining a section 

Where are the sections declared? If you look at a standard C program you won't find any reference to a section. However, if you look at the assembly version of the C program you will find several assembly directives that define the beginning of a section. More precisely, the "`.text`", "`.data`", and "`.section rodata`" directives identify the beginning of the the three canonical sections mentioned previously, while the "`.comm `" directive defines an area of uninitialized data. 

The GNU C compiler translates a source file into the equivalent assembly language file. The next step is carried out by the GNU assembler, which produces an object file. This file is an _ELF relocatable_ file which contains only sections (segments which have absolute addresses cannot be defined in a relocatable file). Sections are now filled, with the exception of the `.bss` section, which just has a length associated with it. 

The assembler scans the assembly lines, translates them into binary code, and inserts the binary code into sections. Each section has its own offset which tells the assembler where to insert the next byte. The assembler acts on one section at a time, which is called the _current section_. In some cases, for instance to allocate space to uninitialized global variables, the assembler does not add bytes in the current section, it just increments its offset. 

> ![\[Assembling sections\]](https://static.lwn.net/images/2013/FigSections.png)

Each assembly language program is assembled separately; the assembler assumes thus that the starting address of an object program is always 0. The GNU linker receives as input a group of these object files and combines them into a single executable file. This kind of linkage is called _static linkage_ because it is performed before running the program. 

The linker relies on a linker script to decide which address to assign to each section of the executable file. To get the default script of your system, you can issue the command: 

```
ld --verbose
```

####  Special sections 

If you compare the sections present in a simple executable file, say one associated with `helloworld.c`, with those present in the Linux kernel executable, you will notice that Linux relies on many _special sections_ not present in conventional executable files. The number of such sections depends on the hardware platform. On an x86_64 system over 30 special sections are defined, while on an ARM system there are about ten. 

You can use the `readelf` command to extract data from the ELF header of `vmlinux`, which is the kernel executable. When issuing this command on an x86_64 box you get something like: 

```
Elf file type is EXEC (Executable file)
        Entry point 0x1000000
        There are 6 program headers, starting at offset 64
    
        Program Headers:
          Type           Offset             VirtAddr           PhysAddr
    		     FileSiz            MemSiz              Flags  Align
          LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
    		     0x00000000007a3000 0x00000000007a3000  R E    200000
          LOAD           0x0000000000a00000 0xffffffff81800000 0x0000000001800000
    		     0x00000000000c7b40 0x00000000000c7b40  RW     200000
          LOAD           0x0000000000c00000 0xffffffffff600000 0x00000000018c8000
    		     0x0000000000000d60 0x0000000000000d60  R E    200000
          LOAD           0x0000000000e00000 0x0000000000000000 0x00000000018c9000
    		     0x0000000000010f40 0x0000000000010f40  RW     200000
          LOAD           0x0000000000eda000 0xffffffff818da000 0x00000000018da000
    		     0x0000000000095000 0x0000000000163000  RWE    200000
          NOTE           0x0000000000713e08 0xffffffff81513e08 0x0000000001513e08
    		     0x0000000000000024 0x0000000000000024         4
    
         Section to Segment mapping:
          Segment Sections...
           00     .text .notes __ex_table .rodata __bug_table .pci_fixup __ksymtab 
    	      __ksymtab_gpl __ksymtab_strings __init_rodata __param __modver 
           01     .data 
           02     .vsyscall_0 .vsyscall_fn .vsyscall_1 .vsyscall_2 .vsyscall_var_jiffies 
    	      .vsyscall_var_vgetcpu_mode .vsyscall_var_vsyscall_gtod_data 
           03     .data..percpu 
           04     .init.text .init.data .x86_trampoline .x86_cpu_dev.init .altinstructions 
    	      .altinstr_replacement .iommu_table .apicdrivers .exit.text .smp_locks 
    	      .data_nosave .bss .brk 
           05     .notes
```

####  Defining a Linux special section 

Special sections are defined in the _Linux linker script_ , which is a linker script distinct from the default linker script mentioned above. The corresponding source file is stored in the `kernel/vmlinux.ld.S` in the architecture-specific subtree. This file uses a set of macros defined in the `linux/include/asm_generic/vmlinux.lds.h` header file. 

The linker script for the ARM hardware platform contains an easy-to-follow definition of a special section: 

```
. = ALIGN(4);
        __start___ex_table = .;
        *(__ex_table)
        __stop___ex_table = .;
```

The `__ex_table` special section is aligned to a multiple of four bytes. Furthermore, the linker creates a pair of identifiers, namely `__start___ex_table` and `__stop___ex_table`, and sets their addresses to the beginning and the end of `__ex_table`. Linux functions can use these identifiers to iterate through the bytes of `__ex_table`. Those identifiers must be declared as `extern` because they are defined in the linker script. 

Defining and using special sections can thus be summarized as follows: 

  * Define the special section "`.special`" in the Linux linker script together with the pair of identifiers that delimit it. 

  * Insert the `.section .special` assembly directive into the Linux code to specify that all bytes up to the next `.section` assembly directive must be inserted in `.special`. 

  * Use the pair of identifiers to act on those bytes in the kernel. 

This technique seems to apply to assembly code only. Luckily, the GNU C compiler offers the non-standard `attribute` construct to create special sections. The 

```
__attribute__((__section__(".init.data")))
```

declaration, for instance, tells the compiler that the code following that declaration must be inserted into the `.init.data` section. To make the code more readable, suitable macros are defined. The `__initdata` macro, for instance, is defined as: 

```
#define __initdata __attribute__((__section__(".init.data")))
```

####  Some examples 

As seen in the previous `readelf` listing, all special sections appearing in the Linux kernel end up packed in one of the segments defined in the `vmlinux` ELF header. Each special section fulfills a particular purpose. The following list groups some of the Linux special sections according to the type of information stored in them. Whenever applicable, the name of the macro used in the Linux code to refer to the section is mentioned instead of the special section's name. 

  * Binary code 

Functions invoked only during the initialization of Linux are declared as `__init` and placed in the `.init.text` section. Once the system is initialized, Linux uses the section delimiters to release the page frames allocated to that section. 

Functions declared as `__sched` are inserted into the `.sched.text` special section so that they will be skipped by the `get_wchan()` function, which is invoked when reading the `/proc/PID/wchan` file. This file contains the name of the function, if any, on which process `PID` is blocked (see [WCHAN the waiting channel ](<http://weichong78.blogspot.it/2006/10/wchan-waiting-channel.html>) for further details). The section delimiters bracket the sequence of addresses to be skipped. The `down_read()` function, for instance, is declared as `__sched` because it gives no helpful information on the event that is blocking the process. 

  * Initialized data 

Global variables used only during the initialization of Linux are declared as `__initdata` and placed in the `.init.data` section. Once the system is initialized, Linux uses the section delimiters to release the page frames allocated to the section. 

The `EXPORT_SYMBOL()` macro makes the identifier passed as parameter accessible to kernel modules. The identifier's string constant is stored in the `__ksymtab_strings` section. 
  * Function pointers 

To invoke an `__init` function during the initialization phase, Linux offers an extensive set of macros (defined in `<linux/init.h>`); `module_init()` is a well-known example. Each of these macros puts a function pointer passed as its parameter in a `.initcall _i_.init` section (`__init` functions are grouped in several classes). During system initialization, Linux uses the section delimiters to successively invoke all of the functions pointed to. 

  * Pairs of instruction pointers 

The `_ASM_EXTABLE(addr1, addr2)` macro allows the page fault exception handler to determine whether an exception was caused by a kernel instruction at address `addr1` while trying to read or write a byte into a process address space. If so, the kernel jumps to `addr2` that contains the _fixup code_ , otherwise a kernel oops occurs. The delimiters of the `__ex_table` special section (see the previous linker script example) set the range of critical kernel instructions that transfer bytes from or to user space. 

  * Pairs of addresses 

The `EXPORT_SYMBOL()` macro mentioned earlier also inserts in the `ksymtab` (or `ksymtab_gpl`) special section a pair of addresses: the identifier's address and the address of the corresponding string constant in `ksymtab` (or `ksymtab_gpl`). When linking a module, the special sections filled by `EXPORT_SYMBOL()` allow the kernel to do a binary search to determine whether an identifier declared as `extern` by the module belongs to the set of exported symbols. 

  * Relative addresses 

On SMP systems, the `DEFINE_PER_CPU(type, varname)` macro inserts the `varname` uninitialized global variable of `type` in the `.data..percpu` special section. Variables stored in that section are called _per-CPU variables_. Since `.data..percpu` is stored in a segment whose initial address is set to 0, the addresses of per-CPU variables are relative addresses. During system initialization, Linux allocates a memory area large enough to store the `NR_CPUS` groups of per-CPU variables. The section delimiters are used to determine the size of the group. 

  * Structures 

The kernel's [SMP alternatives](<https://lwn.net/Articles/164121/>) mechanism allows a single kernel to be built optimally for multiple versions of a given processor architecture. Through the magic of boot-time code patching, advanced instructions can be exploited if, and only if, the system's processor is able to execute those instructions. This mechanism is controlled with the `alternative()` macro: 

```
alternative(oldinstr, newinstr, feature);
```

This macro first stores `oldinstr` in the `.text` regular section. It then stores in the `.altinstructions` special section a structure that includes the following fields: the address of the `oldinstr`, the address of the `newinstr`, the `feature` flags, the length of the `oldinstr`, and the length of the `newinstr`. It stores `newinstr` in a `.altinstr_replacement` special section. Early in the boot process, every alternative instruction which is supported by the running processor is patched directly into the loaded kernel image; it will be filled with no-op instructions if need be. 
Additional special sections, besides `__ksymtab` and `__ksymtab_strings`, are introduced to handle modules. Kernel objects of the form `*.ko` have an ELF relocatable format and the ELF header of such files defines a pair of special sections called `.modinfo` and `.gnu.linkonce.this_module`. Unlike the special sections of the static kernel, these two sections are "address-less" because kernel objects do not contain segments. 

The `.modinfo` section is used by the `modinfo` command to show information about the kernel module. The data stored in the section is not loaded in the kernel address space. The `.gnu.linkonce.this_module` special section includes a `module` structure which contains, among other fields, the module's name. When inserting a module, the `init_module()` system call reads the `module` structure from this special section into an area of dynamic memory. 

####  Conclusion 

Although special sections can be defined in application programs too, there is no doubt that kernel developers have been quite creative in exploiting them. In fact, the examples listed above are by no means exhaustive and new special sections keep popping up in recent kernel releases. Without special sections, implementing some kernel features like those above would be rather difficult.
