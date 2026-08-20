---
title: The first half of the 7.2 merge window
url: https://lwn.net/Articles/1078068/
date: "June 18, 2026"
category: "Releases-7.2"
author: "By Jonathan Corbet June 18, 2026"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 18, 2026

The 7.2 merge window started with the [7.1 kernel release](<https://lwn.net/ml/all/CAHk-=wi4BF4bMhZNZ1tqs+FFV4OuZRe3ZqdWB+LxRLmRweUzQw@mail.gmail.com/>) on June 14\. As of this writing, just over 7,000 non-merge changesets have been pulled into the mainline for the next kernel release. Many of the core subsystems have been pulled at this point, meaning that most of the changes that can be expected in 7.2 have now come into focus. 

The most significant changes merged so far include: 

#### Architecture-specific

  * More i486-specific support has been removed from the kernel; among other things, this includes the removal over 13,000 lines of code needed to emulate the floating-point unit on processors that lacked one. 
  * The AMD Geode processor, used in the [OLPC XO-1](<https://en.wikipedia.org/wiki/OLPC_XO>) computer, has been [marked as orphaned](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4af2468b82bd>). 
  * The Intel [Trusted Domain Extensions (TDX)](<https://www.intel.com/content/www/us/en/developer/tools/trust-domain-extensions/overview.html>) feature is implemented as a special software module loaded from flash memory. Changes in 7.2 include the introduction of a new device type for the management of this module and the ability to replace the module (to install security updates, for example) in a running system. [This documentation patch](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2b9ad7a6154e>) contains some more information. 
  * The s390 architecture has gained Rust support. 
  * The arm64 kernel is being changed to remove its data regions from the all-of-memory linear map as a hardening tactic. While the necessary work has been done, the actual change to effect the removal [has been reverted](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f102131c842d>) for now, pending a solution to a KVM regression. 
  * PowerPC was the only architecture in the kernel with a hardware-optimized MD5 hash implementation. Given the weakness of MD5 in general, this code makes little sense and [has been removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cf52058dcdd9>). 

#### BPF

  * BPF programs attached to tracepoints are now allowed to fault, enabling them to reliably access user-space memory. 
  * The [`bpf()`](<https://man7.org/linux/man-pages/man2/bpf.2.html>) system call has been extended with "common attributes" support. Initially, this feature is used to control logging for a number of BPF subcommands; see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6318f11d53a3>) for (some) more information. 
  * The limit of five parameters to BPF functions and kfuncs has been lifted by allowing additional parameters to be placed on the stack; see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cd59fa185a03>) for more information. 
  * It is now possible for kernel code to safely access [BPF arenas](<https://lwn.net/Articles/961941/>) without having to worry about page faults; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=dc11a4dba246>) for details. 
  * BPF hash maps have a fixed number of buckets set at creation time. The new resizable hash map type removes that limitation; see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=87d119abc42f>) for more information. 
  * It is now possible to quickly attach the same BPF program to multiple tracepoints. Documentation for this feature is absent, but [the patch series](<https://lwn.net/ml/bpf/20260603110554.29590-1-jolsa@kernel.org/>) includes a number of self tests showing how the feature can be used. 

#### Core kernel

  * The implementation of `/proc/interrupts` has been reworked for better performance; see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=13e1a6d6a17e>) for more information. ""/proc/interrupts was subject to micro optimizations for a long time, but most of the low hanging fruit was left on the table"". 
  * The [cache-aware load-balancing](<https://lwn.net/Articles/1018334/>) patch series has been merged. With these changes, the scheduler tries to group processes that share resources into the same cache domain, with, hopefully, an increase in performance. 
  * The addition of support for [sub-schedulers](<https://lwn.net/Articles/1056014/>) in sched_ext continues; see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5b33fc6492a7>) for the current status. 

#### Filesystems and block I/O

  * Filesystems can now provide information about their case-sensitivity via the [`file_getattr()` system call](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=662416578541>). Two new flags are implemented: `FS_XFLAG_CASEFOLD` indicates that file-name lookups are case-insensitive, while `FS_XFLAG_CASENONPRESERVING` indicates that the specific case of new file names will not be preserved. Among other things, this feature supports Windows NFS clients, which expect case-insensitive behavior. 
  * The [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>) system call supports a new flag called `O_EMPTYPATH`. It exists for a specific purpose: to allow a process to reopen an `O_PATH` file descriptor to gain access to the file behind it. With `O_EMPTYPATH`, an empty path name can be given to `openat2()`, and the given file descriptor will be used to find the file to (re)open. 
  * Also new to `openat2()` is the `OPENAT2_REGULAR` flag, which will cause an open to fail if the target is not a regular file. The return code in the failure case is a new (to Linux) one: `EFTYPE`. This flag is meant to be a hardening feature to prevent programs from being manipulated into opening special files. 
  * The XFS filesystem has had [support for zoned storage devices](<https://lwn.net/Articles/1001751/>) since the 6.15 release; that feature is [no longer marked experimental](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=610c08cbe79b>) in 7.2. 
  * Btrfs filesystems now use large folios by default. Support for "huge" folios (up to 2MB) has been added as an experimental option. 
  * The new dm-inlinecrypt device-mapper target can support block devices with inline-encryption capability. Documentation can be found in [Documentation/admin-guide/device-mapper/dm-inlinecrypt.rst](<https://docs.kernel.org/next/admin-guide/device-mapper/dm-inlinecrypt.html>). 

#### Hardware support

  * **GPIO and pin control** : Waveshare DSI-panel GPIO controllers. 
  * **Graphics** : Focaltech OTA7290B panels, Novatek NT35532-based DSI video mode panels, and CHIPWEALTH CH13726A-based DSI panels. Meanwhile, the driver for the venerable Hercules ISA graphics card [has been removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=37a91b995952>) after probably not having worked for decades. 
  * **Hardware monitoring** : MPS MP2985 dual loop digital multi-phase controllers, Luxshare LX1308 DC/DC power modules, Analog Devices MAX20830 step-down DC-DC switching regulators, Analog Devices MAX20860A step-down converters, Analog Devices LTC4283 negative voltage hot swap controllers, Delta E50SN12051 power modules, ARCTIC fan controllers, Murata D1U74T power supplies, and Microchip Technology EMC181X/33 multichannel low-voltage remote diode sensors. 
  * **Miscellaneous** : ASPEED AST2700 interrupt controllers, Qualcomm IPQ6018 pulse-width modulators, Loongson-2 Fast Speed I2C adapters, SG Micro SGM3804 voltage regulators, Spacemit K1 SPI controllers, Qualcomm Gunyah watchdog timers, and Andes ATCWDT200 watchdog timers. 
  * **Networking** : Alibaba Elastic Ethernet adapters, NXP NETC Ethernet switches, Airoha AN8801 Gigabit PHYs, and Realtek 8922AU USB wireless network adapters. 
  * **Sound** : Texas Instruments TAS675x quad-channel audio amplifiers, Everest Semi ES9356 codecs, and Cirrus Logic CS42448/CS42888 codecs. 

#### Miscellaneous

  * [Event probes (eprobes)](<https://docs.kernel.org/trace/eprobetrace.html>) can now more easily dereference structure pointers using the kernel's BTF information; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=69efd863a785>) for more information. 

#### Networking

  * The implementation of the [TCP authentication option](<https://datatracker.ietf.org/doc/html/rfc5925>), which was [added to the kernel](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7fe0e38bb669>) for the 6.7 release, has been changed to use the new [libcrypto](<https://origin.kernel.org/doc/html/next/crypto/libcrypto.html>) library, making the code simpler and more efficient. Support for a number of (presumably) unused algorithms has been removed. See [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fdd2c9a1d082>) and [this documentation patch](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=34d67417c8cf>) for more information. 
  * The number of subflows supported for multi-path TCP connections has been increased from eight to 64\. 
  * Work to improve the scalability of the networking stack by reducing use of the `rtnl_lock` ("the big networking lock") continues. 
  * Removals in the network subsystem include support for ISA and PCMCIA-based ARCnet interfaces, PCMCIA-based Bluetooth adapters, Chelsea inline TLS accelerators, more ATM support (see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e1f544466b2c>) for details), and the AppleTalk protocol. 

#### Security-related

  * The slab allocator has gained the ability to use Clang [allocation tokens](<https://clang.llvm.org/docs/AllocToken.html>) to separate the placement of different types of allocated objects. That makes it harder to exploit a buffer overflow for one type of object to corrupt objects of a different type. See [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=feb662d9168b>) for more information. 
  * The `AF_ALG` mechanism, which allows the kernel's cryptographic layer to offload operations to special-purpose hardware, has been implicated in a number of recent security problems. Given that there is rarely reason to use this feature in the first place, it [has been deprecated](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a67afb1884ba>) with an eye toward a future removal. Meanwhile, support for hardware accelerators [has been removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7524070f26d8>), leaving only software implementations available. A number of drivers for random-number generators using the obsolete `rng_alg` framework have also been removed. 

#### Internal kernel changes

  * There is a new `bh_submit()` function to submit a buffer head for I/O, and the `b_end_io` callback pointer has been removed. This change improves both performance and security; see [this merge message](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f0d857543e4d>) for details. 
  * The [iomap](<https://docs.kernel.org/filesystems/iomap/index.html>) layer has gained support for [fs-verity](<https://docs.kernel.org/filesystems/fsverity.html>). This feature was added without the accompanying documentation updates, which must certainly be coming soon. 
  * There _is_ new documentation for aspiring filesystem developers: [Documentation/filesystems/adding-new-filesystems.rst](<https://docs.kernel.org/next/filesystems/adding-new-filesystems.html>) gives a set of considerations and requirements for the addition of new filesystems to the kernel. 
  * The minimum version of LLVM needed to build the kernel is now 17.0.1. 
  * The new, rigorously undocumented `kconfig-sym-check` build target will check `Kconfig` files for references to symbols that are never defined. 
  * The [Rust "zerocopy" crate](<https://docs.rs/zerocopy/latest/zerocopy/>) has been brought into the kernel source. This crate provides low-cost memory-manipulation primitives, with the intent of isolating unsafe code. See [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=98cc68794c00>) for more information. 
  * The new "higher-ranked lifetime types" for Rust code better encapsulate the connection between driver lifetimes and the devices they are bound to; see [this merge commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2c7c65933600>) for more information. 
  * The [nolibc library](<https://lwn.net/Articles/920158/>) now supports the OpenRISC and 32-bit PA-RISC architectures; it has also gained `alloca()`, `assert()`, `creat()`, and `ftruncate()` implementations. 
  * There is a new error-injection framework for testing at the block-layer level; see [Documentation/block/error-injection.rst](<https://docs.kernel.org/next/block/error-injection.html>) for some more information. 

The 7.2 merge window can be expected to remain open through June 28\. There are just over 6,000 changesets sitting in linux-next, meaning that there is still a fair amount of code waiting to move into the mainline. As always, LWN will post a summary of what that code contains after the merge window closes.
