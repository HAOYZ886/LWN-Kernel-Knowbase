---
title: The first half of the 5.2 merge window
url: https://lwn.net/Articles/787963/
date: "May 10, 2019"
category: "Releases-5.2"
author: "By Jonathan Corbet May 10, 2019"
---

By **Jonathan Corbet**  
May 10, 2019

When he [released the 5.1 kernel](<https://lwn.net/Articles/787556/>), Linus Torvalds noted that he had a family event happening in the middle of the 5.2 merge window and that he would be offline for a few days in the middle. He appears to be trying to make up for lost time before it happens: over 8,300 non-merge changesets have found their way into the mainline in the first four days. As always, there is a wide variety of work happening all over the kernel tree. 

#### Architecture-specific

  * On x86-64 systems, crash-dump kernels could only be placed in memory below 896MB; in 5.2, that limit has been removed. This will break with ancient versions of kexec-tools, but it appears that those versions are unable to work with current kernels anyway. 
  * A lot of work has been done to eliminate the final places where the kernel might execute code from a writable mapping, closing a number of potential holes that could be exploited in an attack. 
  * The s390 architecture now supports kernel address-space layout randomization and signature verification in `kexec_file_load()`. 
  * The PA-RISC architecture now supports the KGDB kernel debugger, jump labels, and kprobes. 
  * The MIPS32 architecture has gained a just-in-time compiler for the eBPF virtual machine. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

#### Core kernel

  * The `clone()` system call has a new `CLONE_PIDFD` flag. When it is present, the return value to the parent will be a file descriptor representing the newly created child process; this descriptor (a "pidfd") can be used for [race-free process signaling](<https://lwn.net/Articles/784831/>) among other things. 
  * The kernel now exports the attributes of the memory attached to each node in sysfs, allowing user space to understand how different memory banks on heterogeneous-memory systems will perform. See [this commit](<https://git.kernel.org/linus/e1cf33aafb84>) and [this commit](<https://git.kernel.org/linus/13bac55ef7ae>) for details and documentation. Note that this work appears to be independent of the [heterogeneous memory work](<https://lwn.net/Articles/787326/>) discussed at the 2019 Linux Storage, Filesystem, and Memory-Management Summit. 
  * The BPF verifier has seen some optimization work that yields a 20x speedup on large programs. That has enabled an increase in the maximum program size (for the root user) from 4096 instructions to 1,000,000. 
  * BPF programs may now access global data; see [this commit changelog](<https://git.kernel.org/linus/d859900c4c56>) for some details. 
  * It is now possible to [install a BPF program to control changes to sysctl knobs](<https://lwn.net/Articles/785263/>). See [Documentation/bpf/prog_cgroup_sysctl.rst](<https://lwn.net/Articles/787994/>) for information on the API for these programs. 

#### Filesystems and block layer

  * The XFS filesystem has gained a health-tracking infrastructure and a new `ioctl()` command to query the health status of a filesystem. It has not, however, gained any documentation describing this feature or how to use it. 
  * The BFQ I/O scheduler has seen [another set of significant performance improvements](<https://lwn.net/Articles/784267/>). 
  * The io_uring mechanism has a new operation, `IORING_OP_SYNC_FILE_RANGE`, which performs the equivalent of a [`sync_file_range()`](<http://man7.org/linux/man-pages/man2/sync_file_range.2.html>) system call. It is also now possible to register an eventfd with an io_uring and get notifications when operations complete. 
  * The [new system calls for filesystem mounting](<https://lwn.net/Articles/759499/>) have finally made it into the mainline kernel. [This commit](<https://git.kernel.org/linus/f1b5618e013a>) contains a sample program showing how to use them. 
  * The ext4 filesystem has gained support for [case-insensitive lookups](<https://lwn.net/Articles/784041/>). As part of that work, the kernel now has [generic support for UTF-8 string handling](<https://lwn.net/Articles/784124/>). 
  * The CIFS filesystem now supports the `FIEMAP` `ioctl()` operation for efficient extent mapping. 

#### Hardware support

  * **Counters** : there is now a generic interface for devices that count things; see [this commit](<https://git.kernel.org/linus/09e7d4ed8991>) for interface documentation and [this one](<https://git.kernel.org/linus/ea2b23b89579>) for sysfs documentation. Supported devices include ACCES 104-QUAD-8, STM32 Timer encoders, STM32 LP Timer encoders, and FlexTimer module quadrature decoders. 
  * **Fieldbus** : The kernel now supports the [Fieldbus](<https://en.wikipedia.org/wiki/Fieldbus>) protocol and, in particular, the HMS Anybus-S, Arcx Anybus-S, and HMS Profinet IRT controllers. 
  * **Graphics** : the kernel finally has support for ARM Mali GPUs. Two new drivers have been merged: Lima for older GPUs and Panfrost for the more recent ones. Also added was support for Ronbo Electronics RB070D30 panels, Feiyang FY07024DI26A30-D MIPI-DSI LCD panels, Rocktech JH057N00900 MIPI touchscreen panels, and ASPEED BMC display controllers. 
  * **Industrial I/O** : MaxSonar I2CXL family ultrasonic sensors, Maxim MAX31856 thermocouple temperature sensors, NXP FXAS21002C gyro sensors, and Texas Instruments ADS8344 analog-to-digital converters. 
  * **Input** : users of Logitech devices with non-unifying receivers should notice an improvement of support for various device features; the input layer now interacts directly with the devices rather than relying on the HID emulation in the receiver. 
  * **Miscellaneous** : ARM SMMUv3 performance monitor counter groups, Cirrus Logic Lochnagar2 temperature, voltage and current sensors, Infineon IR38064 voltage regulators, Intersil ISL68137 PWM controllers, Xilinx Zynq quad-SPI controllers, Daktronics KPC DMA controllers, Aspeed ast2400/2500 HOST P2A VGA MMIO to BMC bridges, STMicroelectronics STM32 factory-programmed memory, Texas Instruments LM3532 backlight controllers, Amlogic G12a-based MDIO bus multiplexers, Milbeaut USIO/UART serial ports, SiFive UARTs, STMicroelectronics MIPID02 CSI-2 to parallel bridges, Amlogic Meson G12A AO CEC controllers, and Amazon elastic fabric adapters. 
  * **Networking** : MediaTek HCI MT7663S and MT7668S SDIO Bluetooth interfaces, NXP SJA1105 Ethernet switches, Realtek 802.11ac wireless interfaces, and MediaTek MT7615E wireless interfaces. 
  * **Pin control** : Cirrus Logic Lochnagar pin controllers, Mediatek MT8516 pin controllers, and Bitmain BM1880 pin controllers. 
  * **Sound** : Support for audio devices running Intel's [Sound Open Firmware](<https://lwn.net/Articles/749888/>) has landed in the mainline. Also supported are Microchip inter-IC sound controllers. 
  * **USB** : Broadcom Stingray USB PHYs, Amlogic G12A USB PHYs, MediaTek UFS M-PHYs, Texas Instruments AM654 SerDes PHYs, and Hisilicon HI3660 USB PHYs. 
  * **Note also** that the support for legacy IDE devices has been deprecated, with an eye toward removal in 2021. If there is anybody out there still using IDE devices that have not been converted over to libata support, now is the time to start saying something. 

#### Security-related

  * The new `mitigations=` command-line option provides simplified control over which speculative-execution vulnerability defenses are enabled. Setting it to `off` disables mitigations entirely. The default option of `auto` turns mitigations on, but will not affect whether hyperthreading is enabled; `auto,nosmt` will also disable hyperthreading if a mitigation requires that. 
  * The elliptic curve Russian digital signature algorithm (GOST R 34.10-2012, RFC 7091, ISO/IEC 14888-3) is now supported. 
  * The work to mark all implicit fall-through cases in `switch` statements is [almost complete](<https://git.kernel.org/linus/b4b52b881cf0>) in 5.2, with only 32 cases left to be addressed. Once they are done, it will be possible to enable the `-Wimplicit-fallthrough` option on kernel builds to prevent them from coming back. 

#### Internal kernel changes

  * The `objtool` utility now tracks code that disables supervisor mode access protection (SMAP, which prevents the kernel from accessing user-space data) to ensure that it is re-enabled before calling any other functions. It is relatively easy to end up in surprising parts of the kernel with SMAP disabled, leading to potential security holes; this change should prevent that from happening. 
  * The interrupt and exception stacks on x86-64 systems now have guard pages, allowing stack overflows to be reliably caught and dealt with. The older probabilistic stack-overflow checking option has been removed, since it is no longer needed. 
  * The new `VM_FLUSH_RESET_PERMS` VMA flag will cause the kernel to immediately clear TLB entries and direct-map permissions for memory with execute permissions. This flag can be set with `set_vm_flush_reset_perms()` or at allocation time. 
  * The `mmiowb()` primitive, which inserts a barrier for memory-mapped I/O operations, [has been removed](<https://lwn.net/Articles/780710/>) in favor of infrastructure that handles barriers automatically when they are needed. 
  * The new inode method `free_inode()` serves as a better version of `destroy_inode()` when RCU is involved; see [this commit](<https://git.kernel.org/linus/fdb0da89f4ba>) for some details. Most filesystems have been converted over to this function. 
  * Device tree authors may want to have a look at the new [dos and don'ts document](<https://lwn.net/Articles/787993/>) on how to write bindings. 

The usual schedule would have the 5.2 merge window closing on May 19, with the final 5.2 release happening in the first half of July. It seems like the list of changes for the second half of this merge window will be smaller than the first, but we'll catch up with it regardless once the window has closed.
