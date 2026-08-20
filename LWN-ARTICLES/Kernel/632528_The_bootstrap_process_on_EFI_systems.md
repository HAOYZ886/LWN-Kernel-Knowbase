---
title: The bootstrap process on EFI systems
url: https://lwn.net/Articles/632528/
date: "February 11, 2015"
category: Bootstrap process
author: "February 11, 2015 This article was contributed by Matt Fleming"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

February 11, 2015

This article was contributed by [Matt Fleming](<http://www.codeblueprint.co.uk/>)

Linux has multiple methods of booting on (U)EFI platforms, each with its merits and drawbacks, and it's not always clear when you should use each of them or why. What follows is a whirlwind tour of the evolution of booting Linux on x86 EFI machines, along with a look at the implementations of the various schemes. 

This article won't cover the basics of booting Linux on BIOS, since that's been [covered elsewhere](<http://www.ibm.com/developerworks/library/l-linuxboot/>) . Instead, it will focus on the particulars of booting on EFI and how the state of the art has changed over time to adapt to the needs of the kernel community and its users. 

#### Legacy EFI boot

Up until Linux 3.3, the only way to boot Linux on EFI hardware was to use an EFI OS loader. This process begins with the EFI firmware boot manager loading a boot loader, such as GRUB. The boot loader is then responsible for loading the Linux kernel and any ramdisk images, taking control of the platform from the firmware, and jumping to the kernel entry point. At a high level, this is all very similar to the way things work on legacy BIOS platforms. In fact, there are very few places where the kernel really knows it was booted from EFI and not BIOS — most of the intricacies of booting on EFI platforms are hidden inside the boot loader. 

If you're trying to bring up Linux on a new firmware, keeping the process as similar as possible to that used on existing, working systems is highly desirable. It achieves maximum code reuse both in terms of boot loader source (since most things just need to be recompiled for the firmware's executable file format) and the difficult-to-debug early kernel boot code. When x86 EFI systems first started hitting the consumer market, making the fewest changes possible was the safest strategy. Of course, there are some special steps that an EFI boot loader must perform, and a handful of EFI-specific pieces of information that need to be passed from the boot loader to the kernel. 

For a start, EFI firmware has two very distinct phases of execution delimited by the termination of the EFI "Boot Services." Initially, the EFI firmware has control over the platform. This allows the firmware to provide a variety of services, such as memory allocation, timers, and event services, to EFI applications and drivers. These are known as Boot Services. It is the boot loader's responsibility to terminate Boot Services by calling the firmware's `ExitBootServices()` function and transition to the second phase of boot (the EFI Runtime Services will remain available). Once `ExitBootServices()` has been called, the firmware relinquishes control of the platform to the OS loader, and ultimately, to the kernel. From that point forward the kernel owns the platform and can install page tables, interrupt handlers, and bring memory allocators online; all the things that usually happen during kernel boot. 

Crucially, some properties of the platform and its devices cannot be discovered post-`ExitBootServices()` because Boot Services functions are required to identify them. The boot loader therefore has the additional responsibility of doing this discovery, stashing the data, and handing it to the kernel inside of a `struct boot_params` object. The relevant members of this structure for EFI boot are highlighted below, 

```
struct boot_params {
    	struct screen_info screen_info;
            ...
    	struct efi_info {
    	    __u32 efi_loader_signature;
    	    __u32 efi_systab;
    	    __u32 efi_memdesc_size;
    	    __u32 efi_memdesc_version;
    	    __u32 efi_memmap;
    	    __u32 efi_memmap_size;
    	    __u32 efi_systab_hi;
    	    __u32 efi_memmap_hi;
    	} efi_info;
    	...
    	__u8  e820_entries;
    	...
    	struct setup_header {
    	    ...
    	    __u64 setup_data;
    	    ...
    	} hdr;
    	...
    	struct e820entry e820_map[E820MAX];
    	...
        };
```

The important pieces of data in this structure are: 

  * The EFI memory map and associated data (found in `efi_info.efi_memmap` and `efi_memmap_hi`). 

  * The EFI System Table pointer (stored in `efi_info.efi_systab` and `efi_systab_hi`). 

  * Graphics (`screen_info`) and PCI device information (`hdr.setup_data`). 

Despite the fact that `struct boot_params` already contains an E820 table (`e820_map`) describing all memory regions, the EFI memory map is also saved and passed to the kernel since it includes EFI-specific region types and other region attributes. The EFI System Table contains data used by the kernel at runtime; for example, it holds a pointer to the Runtime Services dispatch table, which is used by the kernel to do things like create, delete and update EFI variables. The details of any graphics devices present on the platform are stored in `boot_params.screen_info`; key details being the physical address and size of the frame buffer. And finally, PCI device details are recorded in a singly-linked list at `boot_params.hdr.setup_data`. 

Once the boot loader has gathered all of this data, loaded the kernel and optional ramdisk files, and terminated Boot Services, it prepares to jump to the kernel entry point and resume the common boot process. 

With the influx of x86 EFI implementations into the market in 2012, there was a wide variability in their quality, which meant there were a lot of bugs. The most publicized bug was the [Samsung bricking issue](<http://www.pcworld.com/article/2026807/booting-linux-via-uefi-can-brick-some-samsung-laptops.html>), which affected some Samsung laptop models and, once encountered, resulted in users being entirely unable to boot their machines. Some bugs were worked around inside the kernel without too much problem, some were handled in the boot loader. But one thing was clear — the situation was not going to improve overnight. 

Furthermore, because it was impossible to guess at the kind of bugs that would be seen in the future, a more robust solution was needed — one that would not require kernel and boot loader support to be developed in lockstep. The kernel developers needed a way to make changes quickly as new bugs were discovered. A solution was [proposed](<http://marc.info/?l=linux-kernel&m=130756991102498&w=3>) by H. Peter Anvin, and immediately backed by Linus: the kernel would _become_ the boot loader. 

#### EFI boot stub (CONFIG_EFI_STUB)

Peter's idea was to carry a "boot stub" in the kernel image that would take on the responsibility of terminating Boot Services. In some scenarios, this obviates the need for a Linux boot loader entirely. The major benefit that comes with giving the kernel the responsibility for calling `ExitBootServices()` is that it gains control of the platform much earlier than is otherwise possible. That means it can apply bug workarounds much sooner in the boot process. Furthermore, distributing workarounds for firmware bugs to end users now only requires a kernel upgrade, instead of a boot loader upgrade, because the boot ABI would remain unchanged. 

The EFI boot stub doesn't actually live in the kernel proper. Rather, it is a small piece of code that is prepended to a compressed kernel image, and is executed directly by the boot loader or the firmware Boot Manager. So that EFI firmware understands how to load and execute a kernel with the EFI boot stub, a Portable Executable (PE) image header is inserted into the kernel image at build time. To the firmware, the kernel appears to be a legitimate EFI application, complete with entry point address, section tables and relocation information — the kernel image is said to "masquerade" as an EFI application. 

As luck would have it, the only important fields in the PE header that overlap with the `bzImage` header found at the start of the kernel image are the "MZ" magic string (offset 0x0) and the field that points to the rest of the PE header (offset 0x3c). These fields can be inserted into the existing `bzImage` header without causing trouble because the `bzImage` uses those addresses to store code that is only ever executed by unsupported legacy boot loaders to print an error message. The rest of the PE header, pointed at by offset 0x3c, is located at an arbitrary position within the first 512 bytes of the image and contains information such as the EFI boot stub entry point, the architecture the kernel was compiled for, number of sections, size of the image, and other miscellaneous items used in describing a PE application. 

The entry point for the EFI boot stub conforms to the ABI defined in the EFI specification: 

```
EFI_STATUS efi_main(EFI_HANDLE handle, EFI_SYSTEM_TABLE *table);
```

Once the firmware loads the kernel image and jumps to the EFI boot stub entry point, the boot stub begins performing all the necessary jobs for the OS to take control of the platform, just like the boot loader in the legacy EFI boot. 

First, this involves building a `struct boot_params` object, loading any ramdisks specified on the command line, discovering the platform topology and stashing the information for later processing by the kernel. Secondly, `ExitBootServices()` is called and the kernel takes control of the platform. Because all these steps are performed inside the boot stub, it entirely replaces the need for a Linux boot loader. 

Despite the fact that boot loader functionality needs to be re-implemented in the boot stub, the size of the code is actually fairly small. The boot stub uses the Boot Services provided by the firmware for things like memory allocation, filesystem access and printing to the console; the boot stub does not include any drivers of its own. 

This lack of drivers highlights one of the drawbacks of booting using the EFI boot stub. Remember, the boot stub isn't part of the kernel proper, so no matter which drivers you build into your kernel, the boot stub can't use them to load files from Linux filesystems. For most platforms, this ultimately means it is not possible to boot Linux directly from Linux filesystems. Instead, the kernel image and any ramdisk images need to reside on media that the firmware contains built-in drivers for. Essentially, this means the EFI System Partition (ESP) or some other FAT filesystem. 

Additionally, because the PE header contains an architecture field, it's not possible to boot a 64-bit kernel image on 32-bit firmware using the EFI boot stub, since the firmware PE image loader will complain that the file has an incompatible architecture. While these restrictions are acceptable in many situations, some users want to take advantage of the EFI boot stub, but not at the expense of sacrificing a polished boot experience. 

#### EFI handover protocol

The EFI handover protocol first appeared in Linux 3.6. It provides a middle ground by combining the best of the legacy and EFI boot stub schemes: being able to boot from Linux filesystems while applying workarounds in the boot stub. This protocol defines a new entry point into the kernel image — one that points after the code to allocate a `boot_params` structure in the EFI boot stub. The EFI handover protocol cannot be used by EFI firmware because the entry point does not use the EFI application ABI, so a Linux boot loader is required. 

Internally, jumping to the EFI handover protocol entry point still handles early device discovery, saving of the EFI memory map data, and termination of the Boot Services. Since entering the handover entry point skips the `struct boot_params` initialization and loading of ramdisk files, the boot loader is responsible for allocating the object and accessing the filesystem. In the process, a Linux boot loader can provide the features that many users have come to expect: loading from Linux filesystems, graphical boot menus, configuration files, etc. 

A Linux boot loader finds the entry point for the handover protocol by looking at the `handover_offset` field of the `boot_params.hdr` kernel header: 

```
struct boot_params {
    	...
    	struct setup_header {
    	    ...
    	    __u32	handover_offset;
    	} hdr;
        	...
        };
```

The value of this field is an offset from the beginning of the kernel image, and the runtime entry point address can be calculated by adding `boot_params.hdr.handover_offset` to the kernel image load address. 

The entry point prototype looks like this: 

```
void handover(EFI HANDLE handle, EFI_SYSTEM_TABLE *table,
        	 	  struct boot_params *bp);
```

The handover protocol can be thought of as a subset of the EFI boot stub, since code executed by the handover protocol is also executed by the EFI boot stub (but not vice versa). 

#### Summary

Each of the schemes discussed above solves a different problem. If you need the ability to boot from Linux filesystems, using a Linux boot loader such as GRUB with the EFI handover protocol would be the best option since the advantage of using the latest kernel workarounds for firmware bugs is retained. If you just need to load a kernel quickly with no frills, look at booting with the EFI boot stub directly from the firmware. 

It's possible to pass the usual kernel parameters directly to the EFI boot stub, but most are silently passed through to the kernel proper. However, the `initrd=` option is interpreted directly by the EFI boot stub and allows initial ramdisks to be loaded from the ESP and other firmware-supported filesystems. 

Support for building the EFI boot stub is enabled by turning on the `CONFIG_EFI_STUB` option in the kernel config. [Gummiboot](<http://freedesktop.org/wiki/Software/gummiboot/>) is an EFI application loader that leverages kernels built with `CONFIG_EFI_STUB`, and there is no Linux-specific boot code to be found anywhere in the source. It uses the the `LoadImage()` and `StartImage()` Boot Services provided by the firmware to load and run kernel images. I also use `CONFIG_EFI_STUB` kernels containing an embedded ramdisk to boot bare-metal EFI machines. 

The EFI handover protocol entry point is exported when the `CONFIG_EFI_STUB` kernel option is enabled. The versions of GRUB 2 shipping with major distributions (Fedora, Ubuntu, Debian) include support for booting via the EFI handover protocol. Use of this protocol is enabled with the "`linuxefi`" GRUB command (with corresponding `initrdefi` commands for initial ramdisk files), which are used in the same way as the legacy `linux` (and `initrd`) commands, e.g. 

```
linuxefi  /boot/efi/EFI/vmlinuz.efi
        initrdefi /boot/efi/EFI/initramfs.img
```

The handover protocol is also used by [Tianocore's OVMF firmware](<http://www.tianocore.org/ovmf/>) for QEMU when using the `-kernel` command line parameter to directly boot a kernel, e.g. 

```
qemu-system-x86-64 -kernel /path/bzImage
```

Whichever option you pick, the beauty is that a single kernel image can be built that supports all of the schemes.
