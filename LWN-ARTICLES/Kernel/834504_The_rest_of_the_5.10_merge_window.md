---
title: The rest of the 5.10 merge window
url: https://lwn.net/Articles/834504/
date: "October 26, 2020"
category: "Releases-5.10"
author: "By Jonathan Corbet October 26, 2020"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 26, 2020

Linus Torvalds [released 5.10-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=whcRFYSm0jj3Xh3xCyBaxCHA1ZMNO0h_gZso_WZFDUtiQ@mail.gmail.com/>) and closed the 5.10 merge window on October 25; by that time, 13,903 non-merge changesets had been pulled into the mainline repository. Of those, over 6,700 were merged since [LWN's summary](<https://lwn.net/Articles/834157/>) of the first half of the merge window. A fair number of interesting features found their way into the kernel among those commits; read on to catch up with what's coming in 5.10. 

#### Architecture-specific

  * The s390 architecture has never quite gotten the hang of leap seconds; when a leap second happens, the system must be rebooted to make its time reflect the new reality. This, as noted in [this commit](<https://git.kernel.org/linus/b2539aa0d7ff>), ""is not desired"". As of 5.10, s390 systems will be able to adjust to leap seconds on the fly. 
  * The MIPS architecture can now boot kernels compressed with the zstd algorithm. 
  * The RISC-V architecture has gained support for booting on systems with EFI firmware. 
  * Support for non-devicetree i.MX platforms has been removed after having seen no activity for some years. 

#### Core kernel

  * Some types of BPF programs [can now sleep](<https://lwn.net/Articles/825415/>) during their execution. This feature is limited to tracing and security-module programs for now. 
  * The new `BPF_PROG_BIND_MAP` command for the [`bpf()`](<https://man7.org/linux/man-pages/man2/bpf.2.html>) system call binds a map to a loaded program; its purpose is to facilitate the storage of metadata that the program involved does not use directly. 
  * BPF programs can now access per-CPU variables with the `bpf_per_cpu_ptr()` and `bpf_this_cpu_ptr()` helper functions. 
  * The `process_madvise()` system call has been added; it allows one process to perform an [`madvise()`](<https://man7.org/linux/man-pages/man2/madvise.2.html>) call on behalf of another. `process_madvise()` was covered in [this January 2020 article](<https://lwn.net/Articles/810076/>) but the API has changed since then; see [this commit](<https://git.kernel.org/linus/ecb8ac8b1f14>) for the merged version. 

#### Filesystems and block I/O

  * The overlay filesystem has a new "volatile" mode that causes it to ignore all forms of [`fsync()`](<https://man7.org/linux/man-pages/man2/fsync.2.html>) calls. That is, of course, a dangerous mode to operate in, but it is evidently helpful for tasks like image builds where, should the system die in the middle, one can just start over. See [this commit](<https://git.kernel.org/linus/c86243b090bc>) for some details. 
  * The [zonefs filesystem](<https://lwn.net/Articles/794364/>) has gained a new `explicit-open` mount option. If that option is present, opening a file for writing will force the associated zone to be made active on the device. This guarantees that a zone that is successfully opened can be successfully written to later. [This documentation patch](<https://git.kernel.org/linus/48bfd5c6fac1>) has a little more information. 
  * The XFS V4 filesystem format has been deprecated; users are expected to upgrade to the more-capable V5 format. That said, V4 will be supported until at least 2030, so users have some time to make this change. 
  * The ext4 filesystem has a new "fast commits" mode that can significantly reduce the latency of many file operations. The claimed performance improvements are large; see [this patch posting](<https://lwn.net/ml/linux-ext4/20201015203802.3597742-1-harshadshirwadkar@gmail.com/>) for some benchmark results along with a discussion of the fast commits feature in general. 
  * The new `nosymfollow` mount option prevents path resolution from following symbolic links on the mounted filesystem. This option does not prevent applications from using [`readlink()`](<https://www.man7.org/linux/man-pages/man2/readlink.2.html>) and following symbolic links themselves, though. 

#### Hardware support

  * **Clock** : Micro Crystal RV3032 realtime clocks, R-Car V3U clocks, Allwinner A100 clock control units, MediaTek MT8167 clock controllers, and Qualcomm SM8150 and SM8250 display clock controllers. 
  * **Miscellaneous** : Vivaldi keyboards, Renesas RPC-IF HyperBus controllers, Ricoh RN5T618 charger/fuel gauges, TI BQ25980 battery chargers, Mellanox BlueField I2C controllers, joysticks connected via analog-to-digital converters, Zinitix touchscreens, Toshiba Visconti watchdog timers, and TI R5F remote processor subsystems. 
  * **Networking** : MediaTek MT7531 Ethernet switches, Marvell Prestera switches, and Microchip MCP25xxFD SPI CAN controllers. 

#### Networking

  * It is now possible to load a BPF program that can modify TCP header options on packets as they pass through the system. See the changelog in [this commit](<https://git.kernel.org/linus/0813a841566f>) for some information. 
  * The merging of [multipath TCP](<https://lwn.net/Articles/544399/>) support continues; 5.10 will have the ability to transmit data on multiple flows simultaneously. 
  * The IGMPv3/MLDv2 multicast protocol ([RFC 4604](<https://tools.ietf.org/html/rfc4604>)) is now supported. 
  * The ISO 15765-2:2016 CAN transport protocol is now supported. 

#### Security-related

  * The [SafeSetID security module](<https://www.kernel.org/doc/html/latest/admin-guide/LSM/SafeSetID.html>) has gained the ability to control group-ID changes as well. 

#### Virtualization and containers

  * The KVM hypervisor can now defer to a user-space process to handle accesses to unknown model-specific registers (MSRs). See [this commit](<https://git.kernel.org/linus/1ae099540e8c>) for some more information and [this commit](<https://git.kernel.org/linus/1a155254ff93>) for a filtering mechanism that gives more control over MSR handling. 

#### Internal kernel changes

  * The [contiguous memory allocator](<https://lwn.net/Articles/486301/>) has gained optional NUMA awareness; using it requires setting the `DMA_PERNUMA_CMA` configuration option and booting with the `cma_pernuma=` command-line option to specify the size of the per-NUMA space. The DMA mapping layer has been updated to use this feature if it is enabled. 
  * There is a new API for allocating non-coherent DMA areas; see [this documentation patch](<https://git.kernel.org/linus/0d71675f87dc>) for more information. There is also a new function (`dma_direct_alloc_pages()`) for obtaining DMA-addressable memory directly from the page allocator. 
  * Changes have been made to `prandom_u32()`, as [discussed in August](<https://lwn.net/Articles/828705/>), to address some theoretical security issues there. The new code uses a variant of the SipHash hashing function to generate pseudo-random numbers and has added some internal entropy sources. 

Now the time has come to find and fix the remaining bugs in all of that code. That process will continue over the next seven or eight weeks, culminating on a final 5.10 release on December 13 or 20\. As the final release for 2020, 5.10 will probably become the next long-term-support kernel as well so, in a real sense, the work on 5.10 will only be beginning when that release happens in December.
