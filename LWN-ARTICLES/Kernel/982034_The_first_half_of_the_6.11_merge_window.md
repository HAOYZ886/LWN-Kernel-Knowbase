---
title: The first half of the 6.11 merge window
url: https://lwn.net/Articles/982034/
date: "July 18, 2024"
category: "Releases-6.11"
author: "By Jonathan Corbet July 18, 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 18, 2024

The merge window for the 6.11 kernel release opened on July 14; as of this writing, 4,072 non-merge changesets have been pulled into the mainline repository since then. This merge window, in other words, is just now beginning. Still, there has been enough time for a number of interesting changes to land for the next kernel release; read on for a summary of what has been merged so far. 

Some of the most significant changes in the first half of the 6.11 merge window include: 

#### Architecture-specific

  * The 64-bit Arm architecture now support CPU hotplug on ACPI systems; see [this documentation commit](<https://git.kernel.org/linus/828ce929d1c3>) for more information. 
  * X86 kernels can now run as a guest under AMD's [SEV-SNP](<https://www.amd.com/en/developer/sev.html>) secure encrypted virtualization machinery using a [secure VM service module](<https://lwn.net/Articles/921266/>). 
  * The x86 "fake EFI memory map" feature, which allows the creation of fictional memory-map entries at boot, has been removed. The feature is thought to be unused and is not consistent with confidential-computing configurations; as [the merge message](<https://git.kernel.org/linus/e55037c879a0>) puts it: ""With recent developments regarding confidential VMs and unaccepted memory, combined with kexec, creating a known inaccurate view of the firmware's memory map and handing it to the OS is a feature we can live without, hence the removal." 

#### Core kernel

  * The io_uring subsystem now provides operations implementing [`bind()`](<https://man7.org/linux/man-pages/man2/bind.2.html>) and [`listen()`](<https://man7.org/linux/man-pages/man2/listen.2.html>). 
  * A new set of `ioctl()` operations on the nsfs (namespace) filesystem will perform translations of process and thread-group IDs between PID namespaces. 
  * The pidfd filesystem also supports some new `ioctl()` calls to obtain namespace file descriptors for a process represented by a pidfd. 
  * The [nested bottom-half locking patches](<https://lwn.net/Articles/978189/>) have been merged; this is primarily a latency improvement for realtime kernels, but should bring benefits to other users as well. 
  * BPF enhancements include [a new iterator](<https://git.kernel.org/linus/4665415975b0>) for working through bitmasks, a [notification mechanism](<https://git.kernel.org/linus/3f8fde319524>) to tell user space when a `struct_ops` object has been detached, the [ability](<https://git.kernel.org/linus/49df0019f367>) to place more types (including kptrs) in arrays, and the [resilient split BTF mechanism](<https://git.kernel.org/linus/f6afdaf72af7>) for more reliable type metadata in loadable modules. 

#### Filesystems and block I/O

  * The [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) system call now allows the path argument to be a null pointer when the `AT_EMPTY_PATH` flag is set. In current kernels, instead, the path must be an empty string in this case; allowing null pointers enables the kernel to handle `AT_EMPTY_PATH` calls more efficiently. 
  * The [`open_by_handle_at()`](<https://man7.org/linux/man-pages/man2/open_by_handle_at.2.html>) system call will fail in current kernels if the caller lacks the ability to search the initial mount namespace; that makes it unusable in containers. In 6.11, the permission checks for this system call have been relaxed somewhat for situations where the kernel can convince itself that the caller has proper access to the file in question; see [this changelog](<https://git.kernel.org/linus/620c266f3949>) for more information. 
  * Matching the behavior of most Unix systems, the Linux kernel has traditionally prevented writes to an executable file that is in use by a process somewhere in the system; that is the source of the "text file busy" message that some readers may have seen. This restriction is intended to prevent unpleasant surprises in running programs. Kernel developers [have been phasing out](<https://lwn.net/Articles/866493/>) this restriction for a few years, mostly because it does not really protect anything. As of 6.11, the kernel will no longer prevent writes to busy executable files; see [this changelog](<https://git.kernel.org/linus/2a010c412853>) for a lot more details. 
  * The [`listmount()` and `statmount()`](<https://lwn.net/Articles/950569/>) system calls have been extended in a number of ways. `listmount()` is now able to [list mounts in reverse order](<https://git.kernel.org/linus/d04bccd8c19d>), showing the most recent mounts first. Both system calls will now work in the absence of access to the initial mount namespace, and both can now operate in foreign mount namespaces as well as in the local namespace. `statmount()` has gained the ability to return the options with which a filesystem was mounted. 
  * Support for block drivers written in Rust has been merged; thus far, only the [`null_blk` sample driver](<https://git.kernel.org/linus/bc5b533b91ef>) uses this support. Having this support in the mainline will make the development of actually useful block drivers in Rust easier, though; those can be expected to appear in future kernel releases. 
  * The block subsystem now supports [atomic write operations](<https://lwn.net/Articles/974578/>) that will write either a full set of blocks or none of them. At the user-space level, the new `RWF_ATOMIC` flag for [`pwritev()`](<https://man7.org/linux/man-pages/man2/readv.2.html>) can be used to request atomic behavior. `statx()` has been augmented to provide information about atomic-write capabilities for a given file. [This changelog](<https://git.kernel.org/linus/c34fc6f26ab8>) has some more information. 

#### Hardware support

  * **Clock** : Qualcomm SM8650 camera clock controllers. 
  * **Hardware monitoring** : SPD5118-compliant temperature sensors, Monolithic Power Systems MP2993 dual loop digital multi-phase controllers, Monolithic Power Systems MP9941 digital step-down converters, Monolithic Power Systems MP2891 multi-phase digital VR controllers, and Monolithic Power Systems MP5920 hot-swap controllers. 
  * **Miscellaneous** : ChromeOS Embedded Controller sensors, ChromeOS EC-based charge controllers, Analog Devices AXI PWM generators, [emulated PWM devices](<https://git.kernel.org/linus/7f61257cd6e1>) using a GPIO line, Renesas RZ/G2L USB VBUS regulators, QiHeng Electronics ch341a USB-to-SPI adapters, NXP i.MX8MP AudioMix reset controllers, Turris Omnia MCU controllers, and Loongson3 CPU-frequency controllers. 
  * **Networking** : Realtek RTL8192DU USB wireless network adapters, Renesas Ethernet-TSN interfaces, Vining 800 CAN interfaces, Kvaser USBcan Pro 5xCAN and Mini PCIe 1xCAN interfaces, Tehuti Networks TN40xx 10G Ethernet adapters, Synopsys DesignWare Ethernet XPCS controllers, Airoha SoC Gigabit Ethernet adapters, Broadcom BCM4388 Bluetooth chipsets, and Meta "fbnic" network adapters (see [this article](<https://lwn.net/Articles/969383/>) for some background). 

#### Miscellaneous

  * There is a new power-sequencing subsystem charged with ensuring that a system's devices are brought up in the right order. This subsystem is impeccably undocumented; some information can be found in [this changelog](<https://git.kernel.org/linus/e763c9ec71dd>). 
  * The "sloppy logic analyzer" module can turn a set of GPIO lines into a poor-user's logic analyzer; see [this commit](<https://git.kernel.org/linus/7828b7bbbf20>) for more information. ""Note that this is a last resort analyzer which can be affected by latencies, non-deterministic code paths and non-maskable interrupts. It is called 'sloppy' for a reason. However, for e.g. remote development, it may be useful to get a first view and aid further debugging."" 

#### Networking

  * The new `net.tcp_rto_min_us` sysctl knob can be used to adjust the minimum retransmission timeout for TCP sockets. 
  * The `ethtool` utility has gained the ability for fine-tuning the interrupt configuration for interfaces using [Net DIM](<https://www.kernel.org/doc/html/latest/networking/net_dim.html>). There is some minimal documentation in [this commit](<https://git.kernel.org/linus/f750dfe825b9>). 

#### Internal kernel changes

  * The first changes merged for 6.11 were a new "runtime constant" mechanism added by Linus Torvalds. The idea was to replace variables holding values, specifically the pointer to and size of the directory-entry (dentry) cache, that are determined at boot time and never changed again. By simply stuffing those values directly into the instructions that use them, some overhead (a pointer load and a run-time shift) can be avoided. For heavily used data structures, that optimization can make a measurable difference. 

Naturally, there is no documentation; [this commit](<https://git.kernel.org/linus/e78298556ee5>) contains a dummy implementation for architectures that don't support the feature. The `RUNTIME_CONST()` macro is used to define a variable that will be used as a runtime constant. That variable must be set with `runtime_const_init()`, which will rewrite all instructions using it. There are two accessors: `runtime_const_ptr()` and `runtime_const_shift_right_32()`, providing the operations actually needed for the dentry cache. 
  * After several attempts, there is finally [some documentation](<https://git.kernel.org/linus/a7ca193bc9b6>) for the iomap subsystem. 

If the normal schedule holds (and it has been a long time since it didn't), the 6.11 merge window will close with the 6.11-rc1 release on July 28\. There are still over 8,000 changesets sitting in linux-next, so the list of changes for the next release is far from complete. As always, LWN will be back after the closing of the merge window with a summary of what the second half brings.
