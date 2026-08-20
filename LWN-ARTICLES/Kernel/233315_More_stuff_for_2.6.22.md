---
title: More stuff for 2.6.22
url: https://lwn.net/Articles/233315/
date: "May 9, 2007"
category: "Releases-2.6.22"
author: 
---

As of this writing, the 2.6.22 merge window remains open, with quite a bit of code still expected to be merged. User-visible changes which have gone in include: 

  * The mac80211 (formerly Devicescape) wireless networking stack has finally found its way into the mainline. As of this writing there are no drivers which actually use that stack, but drivers are said to be in the works. 

  * The sysfs representation of i2c devices has changed in ways which could break older tools. In particular, versions of lm_sensors prior to 2.10.3 will have problems. 

  * A number of old USB touchscreen drivers (`itmtouch`, `mtouchusb`, and `touchkitusb`) have been removed in favor of the new `usbtouchscreen` driver. 

  * The x86_64 architecture has gained relocatable kernel support, a necessary feature for those wanting to use the kexec-based crash dump mechanism. 

  * Patching of low-level paravirtualization hooks can be inhibited at boot time with the new `noreplace-paravirt` boot flag. 

  * The `REORDER` configuration option, which would rearrange functions in the kernel binary for optimal performance, has been removed from the x86_64 architecture. 

  * The CIFS filesystem supports IPv6 addresses. There is a new mount option to allow user and group IDs to be overridden. A number of performance improvements for CIFS were also merged. 

  * The kernel virtual machine (KVM) API has seen significant changes. If earlier plans still hold, this should be the last set of incompatible KVM changes. 

  * There is now a framework for supporting the "RF kill" switches (which disable the transmitter) found on many mobile devices. 

  * Support for filesystem "subtypes" has been added. The target here is FUSE-based filesystems, which currently all look the same to the kernel and are hard to specify in `fstab`. Now a FUSE ssh-based filesystem can have the type "`fuse.sshfs`". 

  * Entries in `/proc` now exist to provide position and flags information for all open file descriptors. 

  * There is a new system call: 

```
long utimensat(int dirfd, char *filename, struct timespec *times,
                           int flags);
```

This call allows an application to set the access and modification times for the given `filename` with nanosecond precision. 

  * The device mapper has a new "delay" target which can delay I/O operations; this may seem like a feature of dubious value but it's intended for testing only. 

  * Motorola sysv68 disk partition tables are now supported. 

  * There is a new private futex mechanism which improves scalability by avoiding the shared global namespace. 

  * The PowerPC architecture supports the concept of "slices" - special areas of memory which can have different page sizes. The feature is similar to hugetlbfs, but with more page size flexibility. 

  * New hardware supported includes Picotux 200 ARM boards, ADS7846 touchscreen devices, D-Link DSM-G600 boards, MIPS RM9122 integrated serial ports, PMC-Sierra MSP71xx serial devices, MS7712SE01 boards, L-BOX RE2 router boards, SH7780 and SH7722 Solution Engine boards, Sun XVR-500 and XVR-2500 framebuffers, SUN4U PCI-E controllers, Apple system management controllers, Ricoh RS5C313 clock chips, Maxim DS1WM one-wire ASIC cores, Alchemy au1500 programmable serial controllers, Intel LE80578-based framebuffers, PowerPC 750 "Holly" platforms, PowerPC 440GP "Ebony" reference boards, Maxim MAX6650 and MAX6651 fan controllers, Analog Devices AD741x monitoring chips, Intel Core temperature sensors, PA Semi PA6T-1682M random number generators, VIA VT8623 framebuffers, and various drivers for the new "Blackfin" architecture. 

Changes visible to kernel developers include: 

  * The i2c layer has seen significant new changes meant to make i2c drivers look more like drivers for other buses. There are, for example, new `probe()` and `remove()` methods for notifying devices when i2c peripherals come and go. Since i2c is not a self-describing bus, the support code still needs help to know where i2c devices might be; for many classes of device, this information can be had from the system BIOS. 

  * The crypto API has a new set of functions for use with asynchronous block ciphers. There is also a new `cryptd` kernel thread which can run any synchronous cipher in an asynchronous mode. 

  * The `subsystem` structure has been removed from the Linux device model; there never really was any need for it. Most code which was expecting a `struct subsystem` argument has been changed to use the relevant `kset` instead. 

  * There is a new version of the in-kernel rpcbind (portmapper) client which supports versions 2-4 of the rpcbind protocol. The portmapper API has changed as a result. 

  * Numerous changes to the `paravirt_ops` methods have been made. Additionally, `paravirt_ops` is no longer a GPL-only export. 

  * There is a new memory function: 

```
void *krealloc(const void *p, size_t new_size, gfp_t flags);
```

As one would expect, it changes the size of the allocated memory, moving it if need be. 

  * The [SLUB allocator](<http://lwn.net/Articles/229984/>) has been merged as an experimental (for now) alternative to the slab code. 

  * A new macro has been added to make the creation of slab caches easier: 

```
struct kmem_cache KMEM_CACHE(struct-type, flags);
```

The result is the creation of a cache holding objects of the given `struct_type`, named after that type, and with the additional slab `flags` (if any). 

  * The `SLAB_DEBUG_INITIAL` flag has been removed, along with the associated `SLAB_CTOR_VERIFY` flag passed to constructors. The result is a set of changes which ripples through quite a few source files. The unused `SLAB_CTOR_ATOMIC` flag is also gone. 

  * The "quicklist" mechanism has been merged. Quicklists are a simple lookaside cache for page table pages which optimize the allocation and initialization of those pages. 

  * The SuperH architecture has working kgdb support again. 

  * The ia64 architecture has a new tool which will inject machine check errors into a running system. Not recommended for production machines. 

  * The [deferrable timers patch](<http://lwn.net/Articles/228143/>) has been merged. There is also a new macro for initializing workqueue entries (`INIT_DELAYED_WORK_DEFERRABLE()`) which causes the job to be queued in a deferrable manner. 

  * The old `SA_`* interrupt flags have not been removed as originally scheduled, but their use will now generate warnings at compile time. 

  * There is a new `list_first_entry()` macro which, surprisingly, gets the first entry from a list. 

  * The `atomic64_t` and `local_t` types are now fully supported on a wider set of architectures. 

  * The "hibernation" (suspend to disk) code has been separated from the "suspend" (to RAM) code as part of a larger effort to distinguish between those two very different operations. 

  * Workqueues have been reworked again. There is a new function: 

```
void cancel_work_sync(struct work_struct *work);
```

This function tries to cancel a single workqueue entry, be it on the shared (keventd) or a private workqueue. Meanwhile `run_scheduled_work()` has been removed. 

The merging process is not yet done, so expect another big set of patches to go into 2.6.22 before the window closes.
