---
title: "Porting Linux to a new processor architecture, part 2: The early code"
url: https://lwn.net/Articles/656286/
date: "September 2, 2015"
category: "Architectures-Porting to"
author: "September 2, 2015 This article was contributed by Joël Porquet"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

September 2, 2015

This article was contributed by Joël Porquet

In [part 1](<https://lwn.net/Articles/654783/>) of this series, we laid the groundwork for porting Linux to a new processor architecture by explaining the (non-code-related) preliminary steps. This article continues from there to delve into the boot code. This includes what code needs to be written in order to get from the early assembly boot code to the creation of the first kernel thread.

#### The header files

As briefly mentioned in the previous article, the `arch` header files (in my case, located under `linux/arch/tsar/include/`) constitute the two interfaces between the architecture-specific and architecture-independent code required by Linux.

The first portion of these headers (subdirectory `asm/`) is part of the kernel interface and is used internally by the kernel source code. The second portion (`uapi/asm/`) is part of the user interface and is meant to be exported to user space—even though the various standard C libraries tend to reimplement the headers instead of including the exported ones. These interfaces are not completely airtight, as many of the `asm` headers are used by user space.

Both interfaces are typically more than a hundred header files altogether, which is why headers represent one of the biggest tasks in porting Linux to a new processor architecture. Fortunately, over the past few years, developers noticed that many processor architectures were sharing similar code (because they often exhibited the same behaviors), so the majority of this code has been aggregated into a [generic layer of header files](<https://lwn.net/Articles/333569/>) (in `linux/include/asm-generic/` and `linux/include/uapi/asm-generic/`).

The real benefit is that it is possible to refer to these generic header files, instead of providing custom versions, by simply writing appropriate `Kbuild` files. For example, the few first lines of a typical `include/asm/Kbuild` looks like:

```
generic-y += atomic.h
        generic-y += barrier.h
        generic-y += bitops.h
        ...
```

When porting Linux, I'm afraid there is no other choice than to make a list of all of the possible headers and examine them one by one in order to decide whether the generic version can be used or if it requires customization. Such a list can be created from the generic headers already provided by Linux as well as the customized ones implemented by other architectures.

Basically, a specific version must be developed for all of the headers that are related to the details of an architecture, as defined by the hardware or by the software through the ABI: cache (`asm/cache.h`) and TLB management (`asm/tlbflush.h`), the ELF format (`asm/elf.h`), interrupt enabling/disabling (`asm/irqflags.h`), page table management (`asm/page.h`, `asm/pgalloc.h`, `asm/pgtable.h`), context switching (`asm/mmu_context.h`, `asm/ptrace.h`), byte ordering (`uapi/asm/byteorder.h`), and so on.

#### Boot sequence

As explained in part 1, figuring out the boot sequence helps to understand the minimal set of architecture-specific functions that must be implemented—and in which order.

The boot sequence always starts with a function that must be written manually, usually in assembly code (in my case, this function is called `kernel_entry()` and is located in `arch/tsar/kernel/head.S`). It is defined as the main entry point of the kernel image, which indicates to the bootloader where to jump after loading the image in memory.

The following trace shows an excerpt of the sequence of functions that is executed during the boot (starred functions are the architecture-specific ones that will be discussed later in this article):

```
kernel_entry*
        start_kernel
            setup_arch*
            trap_init*
            mm_init
                mem_init*
            init_IRQ*
            time_init*
            rest_init
                kernel_thread
                kernel_thread
                cpu_startup_entry
```

#### Early assembly boot code

The early assembly boot code has this special aura that scared me at first (as I'm sure it did many other programmers), since it is often considered one of the most complex pieces of code in a port. But even though writing assembly code is usually not an easy ride, this early boot code is not magic. It is merely a trampoline to the first architecture-independent C function and, to this end, only needs to perform a short and defined list of tasks.

When the early boot code begins execution, it knows nothing about what has happened before: Has the system been rebooted or just been powered on? Which bootloader has just loaded the kernel in memory? And so forth. For this reason, it is safer to put the processor into a known state. Resetting one or several system registers usually does the trick, making sure that the processor is operating in kernel mode with interrupts disabled.

Similarly, not much is known about the state of the memory. In particular, there is no guarantee that the portion of memory representing the kernel’s `bss` section (the section containing uninitialized data) was reset to zero, which is why this section must be explicitly cleared.

Often Linux receives arguments from the bootloader (in the same way that a program receives arguments when it is launched). For example, this could be the memory address of a [flattened device tree](<http://www.devicetree.org/>) (on ARM, MicroBlaze, openRISC, etc.) or some other architecture-specific structure. Often such arguments are passed using registers and need to be saved into proper kernel variables.

At this point, virtual memory has not been activated and it is interesting to note that kernel symbols, which are all defined in the kernel's virtual address space, have to be accessed through a special macro: [`pa()`](<http://lxr.free-electrons.com/source/arch/x86/kernel/head_32.S?v=4.2#L28>) in x86, [`tophys()`](<http://lxr.free-electrons.com/source/arch/openrisc/kernel/head.S?v=4.2#L32>) in OpenRISC, etc. Such a macro translates the virtual memory address for symbols into their corresponding physical memory address, thus acting as a temporary software-based translation mechanism.

Now, in order to enable virtual memory, a page table structure must be set up from scratch. This structure usually exists as a static variable in the kernel image, since at this stage it is nearly impossible to allocate memory. For the same reason, only the kernel image can be mapped by the page table at first, using huge pages if possible. According to convention, this initial page table structure is called `swapper_pg_dir` and is thereafter used as the reference page table structure throughout the execution of the system.

On many processor architectures, including TSAR, there is an interesting thing about mapping the kernel in that it actually needs to be mapped twice. The first mapping implements the expected direct-mapping strategy as described in part 1 (i.e. access to virtual address `0xC0000000` redirects to physical address `0x00000000`). However, another mapping is temporarily required for when virtual memory has just been enabled but the code execution flow still hasn't jumped to a virtually mapped location. This second mapping is a simple identity mapping (i.e. access to virtual address `0x00000000` redirects to physical address `0x00000000`).

With an initialized page table structure, it is now possible to enable virtual memory, meaning that the kernel is fully executing in the virtual address space and that all of the kernel symbols can be accessed normally by their name, without having to use the translation macro mentioned earlier.

One of the last steps is to set up the stack register with the address of the initial kernel stack so that C functions can be properly called. In most processor architectures (SPARC, Alpha, OpenRISC, etc.), another register is also dedicated to containing a pointer to the current thread's information (`struct thread_info`). Setting up such a pointer is optional, since it can be derived from the current kernel stack pointer (the `thread_info` structure is usually located at the bottom of the kernel stack) but, when allowed by the architecture, it enables much faster and more convenient access.

The last step of the early boot code is to jump to the first architecture-independent C function that Linux provides: `start_kernel()`.

#### En route to the first kernel thread

[`start_kernel()`](<http://lxr.free-electrons.com/source/init/main.c?v=4.2#L497>) is where many subsystems are initialized, from the various virtual filesystem (VFS) caches and the security framework to time management, the console layer, and so on. Here, we will look at the main architecture-specific functions that `start_kernel()` calls during boot before it finally calls `rest_init()`, which creates the first two kernel threads and morphs into the boot idle thread.

#### `setup_arch()`

While it has a rather generic name, `setup_arch()` can actually do quite a bit, depending on the architecture. Yet examining the code for different ports reveals that it generally performs the same tasks, albeit never in the same order nor the same way. For a simple port (with device tree support), there is a simple skeleton that `setup_arch()` can follow.

One of the first steps is to discover the memory ranges in the system. A device-tree-based system can quickly skim through the flattened device tree given by the bootloader (using `early_init_devtree()`) to discover the physical memory banks available and to register them into the `memblock` layer. Then, parsing the early arguments (using `parse_early_param()`) that were either given by the bootloader or directly included in the device tree can activate useful features such as `early_printk()`. The order is important here, as the device tree might contain the physical address of the terminal device used for printing and thus needs to be scanned first.

Next the memblock layer needs some more configuration before it is possible to map the low memory region, which enables memory to be allocated. First, the regions of memory occupied by the kernel image and the device tree are set as being _reserved_ in order to remove them from the pool of free memory, which is later released to the buddy allocator. The boundary between low memory and high memory (i.e. which portion of the physical memory should be included in the direct mapping region) needs to be determined. Finally the page table structure can be cleaned up (by removing the identity mapping created by the early boot code) and the low memory mapped.

The last step of the memory initialization is to configure the memory zones. Physical memory pages can be associated with different zones: `ZONE_DMA` for pages compatible with the old ISA 24-bit DMA address limitation, and `ZONE_NORMAL` and `ZONE_HIGHMEM` for low- and high-memory pages, respectively. Further reading on memory allocation in Linux can be found in [_Linux Device Drivers_ [PDF]](</images/pdf/LDD3/ch08.pdf>).

Finally, the kernel memory segments are registered using the resource API and a tree of [`struct device_node`](<http://lxr.free-electrons.com/source/include/linux/of.h?v=4.2#L49>) entries is created from the flattened device tree.

If `early_printk()` is enabled, here is an example of what appears on the terminal at this stage:

```
Linux version 3.13.0-00201-g7b7e42b-dirty (joel@joel-zenbook) \
            (gcc version 4.8.3 (GCC) ) #329 SMP Thu Sep 25 14:17:56 CEST 2014
        Model: UPMC/LIP6/SoC - Tsar
        bootconsole [early_tty_cons0] enabled
        Built 1 zonelists in Zone order, mobility grouping on.  Total pages: 65024
        Kernel command line: console=tty0 console=ttyVTTY0 earlyprintk
```

#### `trap_init()`

The role of `trap_init()` is to configure the hardware and software architecture-specific parts involved in the interrupt/exception infrastructure. Up to this point, an exception would either cause the system to crash immediately or it would be caught by a handler that the bootloader might have set up (which would eventually result in a crash as well, but perhaps with more information).

Behind (the actually simple) `trap_init()` hides another of the more complex pieces of code in a Linux port: the interrupt/exception handling manager. A big part of it has to be written in assembly code because, as with the early boot code, it deals with specifics that are unique to the targeted processor architecture. On a typical processor, a possible overview of what happens on an interrupt is as follows:

  * The processor automatically switches to kernel mode, disables interrupts, and its execution flow is diverted to a special address that leads to the main interrupt handler.
  * This main handler retrieves the exact cause of the interrupt and usually jumps to a sub-handler specialized for this cause. Often an interrupt vector table is used to associate an interrupt sub-handler with a specific cause, and on some architectures there is no need for a main interrupt handler, as the routing between the actual interrupt event and the interrupt vector is done transparently by hardware.
  * The sub-handler saves the current context, which is the state of the processor that can later be restored in order to resume exactly where it stopped. It may also re-enable the interrupts (thus making Linux re-entrant) and usually jumps to a C function that is better able to handle the cause of the exception. For example, such a C function can, in the case of an access to an illegal memory address, terminate the faulty user program with a `SIGBUS` signal.

Once all of this interrupt infrastructure is in place, `trap_init()` merely initializes the interrupt vector table and configures the processor via one of its system registers to reflect the address of the main interrupt handler (or of the interrupt vector table directly).

#### `mem_init()`

The main role of `mem_init()` is to release the free memory from the memblock layer to the buddy allocator (aka the [page allocator](<https://lwn.net/Articles/320556/>)). This represents the last memory-related task before the slab allocator (i.e. the cache of commonly used objects, accessible via `kmalloc()`) and the vmalloc infrastructure can be started, as both are based on the buddy allocator.

Often `mem_init()` also prints some information about the memory system:

```
Memory: 257916k/262144k available (1412k kernel code, \
            4228k reserved, 267k data, 84k bss, 169k init, 0k highmem)
        Virtual kernel memory layout:
            vmalloc : 0xd0800000 - 0xfffff000 ( 759 MB)
            lowmem  : 0xc0000000 - 0xd0000000 ( 256 MB)
              .init : 0xc01a5000 - 0xc01ba000 (  84 kB)
              .data : 0xc01621f8 - 0xc01a4fe0 ( 267 kB)
              .text : 0xc00010c0 - 0xc01621f8 (1412 kB)
```

#### `init_IRQ()`

Interrupt networks can be of very different sizes and complexities. In a simple system, the interrupt lines of a few hardware devices are directly connected to the interrupt inputs of the processor. In complex systems, the numerous hardware devices are connected to multiple programmable interrupt controllers (PICs) and these PICs are often cascaded to each other, forming a multilayer interrupt network. The device tree helps a great deal by easily describing such networks (and especially the routing) instead of having to specify them directly in the source code.

In `init_IRQ()`, the main task is to call `irqchip_init()` in order to scan the device tree and find all the nodes identified as interrupt controllers (e.g PICs). It then finds the associated driver for each node and initializes it. Unless the targeted system uses an already-supported interrupt controller, that typically means the first device driver will need to be written.

Such a driver contains a few major functions: an initialization function that maps the device in the kernel address space and maps the controller-local interrupt lines to the Linux IRQ number space (through the [`irq_domain`](<https://www.kernel.org/doc/Documentation/IRQ-domain.txt>) mapping library); a mask/unmask function that can configure the controller in order to mask or unmask the specified Linux IRQ number; and, finally, a controller-specific interrupt handler that can find out which of its inputs is active and call the interrupt handler registered with this input (for example, this is how the interrupt handler of a block device connected to a PIC ends up being called after the device has raised an interrupt).

#### `time_init()`

The purpose of `time_init()` is to initialize the architecture-specific aspects of the timekeeping infrastructure. A minimal version of this function, which relies on the use of a device tree, only involves two function calls.

First, `of_clk_init()` will scan the device tree and find all the nodes identified as clock providers in order to initialize the [clock framework](<https://www.kernel.org/doc/Documentation/clk.txt>). A very simple clock-provider node only has to define a fixed frequency directly specified as one of its properties.

Then, `clocksource_of_init()` will parse the clock-source nodes of the device tree and initialize their associated driver. As described in the [kernel documentation](<https://www.kernel.org/doc/Documentation/timers/timekeeping.txt>), Linux actually needs two types of timekeeping abstraction (which are actually often both provided by the same device): a clock-source device provides the basic timeline by monotonically counting (for example it can count system cycles), and a clock-event device raises interrupts on certain points on this timeline, typically by being programmed to count periods of time. Combined with the clock provider, it allows for precise timekeeping.

The driver of a clock-source device can be extremely simple, especially for a memory-mapped device for which the [generic MMIO clock-source driver](<http://lxr.free-electrons.com/source/drivers/clocksource/mmio.c>) only needs to know the address of the device register containing the counter. For the clock event, it is slightly more complicated as the driver needs to define how to program a period and how to acknowledge it when it is over, as well as provide an interrupt handler for when a timer interrupt is raised.

#### Conclusion

One of the main tasks performed by `start_kernel()` later on is to calibrate the number of loops per jiffy, which is the number of times the processor can execute an internal delay loop in one jiffy—an internal timer period that normally ranges from one to ten milliseconds. Succeeding in performing this calibration should mean that the different infrastructures and drivers set up by the architecture-specific functions we just presented are working, since the calibration makes use of most of them.

In the next article, we will present the last portion of the port: from the creation of the first kernel thread to the `init` process.
