---
title: "Understanding the Jailhouse hypervisor, part 1"
url: https://lwn.net/Articles/578295/
date: "January 1, 2014"
category: "Jailhouse; Virtualization-Jailhouse"
author: "January 1, 2014 This article was contributed by Valentine Sinitsyn"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

January 1, 2014

This article was contributed by Valentine Sinitsyn

[Jailhouse](<https://github.com/siemens/jailhouse>) is a new hypervisor designed to cooperate with Linux and run bare-metal applications or modified guest operating systems. Despite this cooperation, Jailhouse is self-contained and uses Linux only to bootstrap and (later) manage itself. The hypervisor is free software released under GPLv2 by Siemens; the Jailhouse project was [publicly announced](<https://lwn.net/Articles/574274/>) in November 2013, and is in an early stage of development. Currently, Jailhouse supports 64-bit x86 systems only; ARM support is on the roadmap, though, and, given that the code is portable, we may see more architectures added to this list in the future. 

Linux has many full-fledged hypervisors (including KVM and Xen), so why bother creating another one? Jailhouse is different. First of all, it is a _partitioning_ hypervisor that is more concerned with isolation than virtualization. Jailhouse is lightweight and doesn't provide many features one traditionally expects from virtualization systems. For example, there is no support for overcommitment of resources, guests can't share a CPU because there is no scheduler, and Jailhouse can't emulate devices you don't have. 

Instead, Jailhouse enables asymmetric multiprocessing (AMP) on top of an existing Linux setup and splits the system into isolated partitions called "cells." Each cell runs one guest and has a set of assigned resources (CPUs, memory regions, PCI devices) that it fully controls. The hypervisor's job is to manage cells and maintain their isolation from each other. This approach is most useful for virtualizing tasks that require full control over the CPU; examples include realtime control tasks and long-running number crunchers (high-performance computing). Besides these, it can be used for security applications: to create sandboxes, for example. 

A running Jailhouse system has at least one cell known as the "Linux cell." It contains the Linux system used to initially launch the hypervisor and to control it afterward. This cell's role is somewhat similar to that of [dom0 in Xen](<http://wiki.xen.org/wiki/Dom0>). However, the Linux cell doesn't assert full control over hardware resources as dom0 does; instead, when a new cell is created, the Linux cell cedes control over some of its CPU, device, and memory resources to that new cell. This process is called "shrinking". 

Jailhouse relies on hardware-assisted virtualization features provided by the target architecture; for Intel processors (the only ones supported as of this writing) this means [VT-x and VT-d](<https://en.wikipedia.org/wiki/X86_virtualization#Intel_virtualization_.28VT-x.29>) support. These requirements make the hypervisor design clean, its code compact and relatively simple; the goal is to keep Jailhouse below 10,000 lines of code. Traditionally, hypervisors were either large and complex, or intentionally simple if built for the classroom. Jailhouse fits in between: it is a real product targeted at production use that is small enough to cover in a two-part article series. 

The easiest way to play with Jailhouse now is to run it inside KVM with a simple bare-metal application, [`apic-demo.bin`](<https://github.com/siemens/jailhouse/blob/master/inmate/apic-demo.c>) (provided with the Jailhouse source), as a guest. In this case, VT-d is not used since KVM doesn't emulate it (yet). The [README file](<https://github.com/siemens/jailhouse/blob/master/README>) describes how to create this setup in detail; additional help can be found in [the mailing list archives](<https://groups.google.com/forum/#!forum/jailhouse-dev>). 

Running Jailhouse on real hardware is also possible, but is not very easy at this time. You will need to describe the resources available to Jailhouse (a process covered in the next section); a good starting point for this is the contents of `/proc/iomem` in your Linux system. This is an error-prone process, but hopefully this article will provide enough insight into how Jailhouse works internally to get it running on the hardware of your choice. 

A good introduction to Jailhouse (including slides) can be found in [the initial announcement](<https://lwn.net/Articles/574273/>). We won't reproduce it here but rather will dive straight into the hypervisor internals. 

#### Data structures

Before it can be used to partition a real system, the Jailhouse system must be told how that system is put together. To that end, Jailhouse uses `struct jailhouse_system` (defined in [`cell-config.h`](<https://github.com/siemens/jailhouse/blob/master/hypervisor/include/jailhouse/cell-config.h>)) as a descriptor for the system it runs on. This structure contains three fields: 

  * `hypervisor_memory`, which defines Jailhouse's location in memory; 

  * `config_memory`, which points to the region where hardware configuration is stored (for x86, it's the ACPI tables); and 

  * `system`, a cell descriptor which sets the initial configuration for the Linux cell. 

A cell descriptor starts with `struct jailhouse_cell_desc`, defined in `cell-config.h` as well. This structure contains basic information like the cell's name, size of its CPU set, the number of memory regions, IRQ lines, and PCI devices. Associated with `struct jailhouse_cell_desc` are several variable-sized arrays which follow immediately after it in memory; these arrays are: 

  * A bitmap which lists the cell's CPUs. 

  * An array which stores the physical address, guest physical address (`virt_start`), size, and access flags for this cell's memory regions. There can be many of these regions, corresponding to the cell's RAM (currently it must be the first region), PCI, ACPI, or I/O APIC, etc. See [`config/qemu-vm.c`](<https://github.com/siemens/jailhouse/blob/master/config/qemu-vm.c>) for an example. 

  * An array which describes the cell's IRQ lines. It's unused now and may disappear or change in the future. 

  * The I/O bitmap, which controls I/O ports accessible from the cell (setting a bit indicates that the associated port is inaccessible). This is x86-only, since no other supported architecture has a separate I/O space. 

  * An array which maps PCI devices to VT-d domains. 

Currently, Jailhouse has no human-readable configuration files. Instead, the C structures mentioned above are compiled with the "`-O binary`" `objcopy` flag to produce raw binaries rather than ELF objects, and the `jailhouse` user-space tool (see [`tools/jailhouse.c`](<https://github.com/siemens/jailhouse/blob/master/tools/jailhouse.c>)) loads them into memory in that form. Creating such descriptors is tedious work that requires extensive knowledge of the hardware architecture. There are no sanity checks for descriptors except basic validation, so you can easily create something unusable. Nothing prevents Jailhouse from using a higher-level XML or similar text-based configuration files in the future — it is just not implemented yet. 

Another common data structure is `struct per_cpu`, which is architecture-specific and defined (for x86) in [`x86/include/asm/percpu.h`](<https://github.com/siemens/jailhouse/blob/master/hypervisor/arch/x86/include/asm/percpu.h>). It describes a CPU that is assigned to a cell. Throughout this text, we will refer to it as `cpu_data`. There is one `cpu_data` structure for each processor Jailhouse manages, and it is stored in a per-CPU memory region called `per_cpu`. `cpu_data` contains information like the logical CPU identifier (`cpu_id` field), APIC identifier (`apic_id`), the hypervisor stack (`stack[PAGE_SIZE]`), a back reference to the cell this CPU belongs to (`cell`), a set of Linux registers (i.e. register values used when Linux moved to this CPU's cell), and the CPU mode (stopped, wait-for-SIPI, etc). It also holds the VMXON and VMCS regions required for VT-x. 

Finally, there is `struct jailhouse_header` defined in [`header.h`](<https://github.com/siemens/jailhouse/blob/master/hypervisor/include/jailhouse/header.h>), which describes the hypervisor as a whole. It is located at the very beginning of the hypervisor binary image and contains information like the hypervisor entry point address, its memory size, page offset, and number of possible/online CPUs. Some fields in this structure have static values, while the loader initializes the others at Jailhouse startup. 

#### Enabling Jailhouse

Jailhouse operates in a physically continuous memory region. Currently, this region must be reserved at boot using the "`memmap=`" kernel command-line parameter; future versions may use the [contiguous memory allocator (CMA)](<https://lwn.net/Articles/486301/>) instead. When you enable Jailhouse, the loader linearly maps this memory into the kernel's virtual address space. Its offset from the memory region's base address is stored in the `page_offset` field of the header. This makes converting from host virtual to physical address (and the reverse) trivial. 

To enable the hypervisor, Jailhouse needs to initialize its subsystems, create a Linux cell according to the system configuration, enable VT-x on each CPU, and, finally, migrate Linux into its cell to continue running in guest mode. From this point, the hypervisor asserts full control over the system's resources. As stated earlier, Jailhouse doesn't depend on Linux to provide services to guests. However, Linux is used to initialize the hypervisor and to control it later. For these tasks, the `jailhouse` user-space tool issues `ioctl()` commands to `/dev/jailhouse`. The `jailhouse.ko` module (the loader), compiled from [`main.c`](<https://github.com/siemens/jailhouse/blob/master/main.c>), registers this device node when it is loaded into the kernel. 

To start the sequence of events described above, the `jailhouse` tool is used to issue a `JAILHOUSE_ENABLE` `ioctl()` which causes a call to `jailhouse_enable()`. It loads the hypervisor code into the reserved memory region via a `request_firmware()` call. Then `jailhouse_enable()` maps Jailhouse's reserved memory region into kernel space using `ioremap()` and marks its pages as executable. The hypervisor and a system configuration (`struct jailhouse_system`) copied from user space are laid out in the reserved region. Finally, `jailhouse_enable()` calls `enter_hypervisor()` on each CPU, passing it the header, and waits until all these calls return. After that, Jailhouse is considered enabled and the firmware is released. 

`enter_hypervisor()` is really a thin wrapper that jumps to the entry point set in the header. The entry point is defined in `[hypervisor/setup.c](<https://github.com/siemens/jailhouse/blob/master/hypervisor/setup.c>)` as `arch_entry`, which is coded in assembler and resides in [`x86/entry.S`](<https://github.com/siemens/jailhouse/blob/master/hypervisor/arch/x86/entry.S>). This code locates the `per_cpu` region for a given `cpu_id`, stores the Linux stack pointer and `cpu_id` in it, sets the Jailhouse stack, and calls the architecture-independent `entry()` function, passing it a pointer to `cpu_data`. When this function returns, the Linux stack pointer is restored. 

The `entry()` function is what actually enables Jailhouse. It behaves slightly differently for the first CPU it initializes than for the rest of them. The first CPU is called "master"; it is responsible for system-wide initialization and checks. It sets up paging, maps `config_memory` if it is present in the system configuration, checks the memory regions defined in the Linux cell descriptor for alignment and access flags, initializes the APIC, creates Jailhouse's Interrupt Descriptor Table (IDT), configures x2APIC guest (VMX non-root) access (if available), and initializes the Linux cell. After that, VT-d is enabled and configured for the Linux cell. Non-master CPUs, instead, only initialize themselves. 

#### CPU initialization

CPU initialization is a lengthy process that begins in the `cpu_init()` function. For starters, the CPU is registered as a "Linux CPU": its ID is validated, and, if it is on the system CPU set, it is added to the Linux cell. The rest of the procedure is architecture-specific and continues in `arch_cpu_init()`. For x86, it saves the current register values in the `cpu_data` structure. These values will be restored on first VM entry. Then Jailhouse swaps the IDT (interrupt handlers), the Global Descriptor Table (GDT) that contains segment descriptors, and CR3 (page directory pointer) register with its own values. 

Finally, `arch_cpu_init()` fills the `cpu_data->apic_id` field (see `apic_cpu_init()`) and configures Virtual Machine Extensions (VMX) for the CPU. This is done in `vmx_cpu_init()`, which first checks that CPU provides all the required features. Then it prepares the Virtual Machine Control Structure (VMCS) which is located in `cpu_data`, and enables VMX on the CPU. The VMCS region is configured in `vmcs_setup()` so that on every VM entry or exit: 

  * The host (Jailhouse) gets the appropriate control and segmentation register values. The corresponding VMCS fields are simply copied from the hardware registers set by `arch_cpu_init()`. The LMA and LME bits are raised in the host's `IA32_EFER` MSR, indicating that the processor is in 64-bit mode, and the stack pointer is set to the end of `cpu_data->stack` (remember that the stack grows down). The host's RIP (instruction pointer) is set to `vm_exit()` defined in `x86/entry.S`, and interrupts are disabled in the host RFLAGS. `vm_exit()` calls `vmx_handle_exit()` function and resumes VM execution with `VMRESUME` instruction when it returns. This way, on each VM exit, interrupts are disabled and control is transferred to the dispatch function that analyzes the exit reason and acts appropriately. SYSENTER MSRs are cleared because Jailhouse has no user-space applications or system calls and its guests use a different means to switch to the hypervisor. 

  * The guest gets its control and segmentation registers from `cpu_data->linux_`*. RSP and RIP are taken from the kernel stack frame created for the `arch_entry()` call. This way, on VM entry, Linux code will continue execution as if the `entry()` call in `hypervisor_enter()` has already completed; thus the kernel is transparently migrated to the cell. The guest's `IA32_EFER` MSR is also set to its Linux value so that 64-bit mode is enabled on VM entry. Cells besides the Linux cell will reset their CPUs just after initialization, overwriting the values defined here. 

When all CPUs are initialized, `entry()` calls `arch_cpu_activate_vmm()`. This is point of no return: it sets the RAX register to zero, loads all the general-purpose registers left and issues a `VMLAUNCH` instruction to enter the guest. Due to the guest register setup described earlier and because RAX (which, by convention, stores function return values) is zero, Linux will consider the `entry()` call to be successful and move on as a guest. 

This concludes the Part 1 of the series. In Part 2, we will look at how Jailhouse handles interrupts, and what needs to be done to create a cell, and to disable the hypervisor.
