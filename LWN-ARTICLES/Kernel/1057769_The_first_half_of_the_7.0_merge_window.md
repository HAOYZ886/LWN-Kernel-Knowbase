---
title: The first half of the 7.0 merge window
url: https://lwn.net/Articles/1057769/
date: "February 13, 2026"
category: "Releases-7.0"
author: "By Daroc Alden February 13, 2026"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Daroc Alden**  
February 13, 2026

The merge window for Linux 7.0 has opened, and with it comes a number of interesting improvements and enhancements. At the time of writing, there have been 7,695 non-merge commits accepted. The 7.0 release is not special, [ according to the kernel's versioning scheme](<http://www.kroah.com/log/blog/2025/12/09/linux-kernel-version-numbers/>) — just the release that comes after 6.19. Humans love symbolism and round numbers, though, so it may feel like something of a milestone. 

The most important changes included in this release were: 

#### Architecture-specific

  * The kernel now [supports](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=58ce78667a64>) atomic 64-byte loads and stores on Arm CPUs that provide the feature. 

#### Core kernel

  * Rust support is officially [ no longer experimental](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=9fa7153c31a3>). Rust is here to stay, although individual subsystem maintainers are still free to keep it out of their subsystems. 
  * BPF can be used [to filter io_uring operations](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=591beb0e3a03>); see [this article](<https://lwn.net/Articles/1054225>) for details. The change adds a way to potentially enforce sandboxing on io_uring operations, given that [ `seccomp()`](<https://www.man7.org/linux/man-pages/man2/seccomp.2.html>) can't block individual io_uring operations — and that therefore administrators with `seccomp()`-based sandboxes typically disable io_uring altogether. 
  * Users have the option of using [non-circular io_uring queues](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5247c034a67f>) for better cache performance in applications where requests are usually completed before the submission system call returns. In a circular queue, the slots where new messages are stored continue advancing in memory until they wrap around. This causes churn in the cache. A non-circular queue will reset the queue's pointers whenever it is empty, hopefully keeping the start of the queue's memory in cache. 
  * Looking up types in BPF type format (BTF) debugging information [now uses](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b9da17391e13>) a binary search, which should make loading BPF programs more efficient. 
  * As [ reported in January](<https://lwn.net/Articles/1055559>), BPF kfuncs can [accept implicit arguments](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b236134f70ba>). 
  * The scheduler has changed to only support [two preemption modes](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7dadeaa6e851>) on most architectures: `PREEMPT_LAZY` and `PREEMPT_FULL`. Only architectures that do not support preemption at all can still configure `PREEMPT_NONE`, and only architectures that don't support lazy preemption can configure `PREEMPT_VOLUNTARY`. See [this article](<https://lwn.net/Articles/944686/>) and its [sequel](<https://lwn.net/Articles/945422/>) for details on the different modes. 
  * The [time-slice extension proposal](<https://lwn.net/Articles/1038235/>) for restartable sequences has [ been merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=36ae1c45b2ce>). This change allows processes that are almost done with a lock at the end of their time slice to request a short grace period to finish their work and release it. 
  * Administrators of systems that need to panic when workqueues stall can set [a new build-time configuration option](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=32d572e39031>) to force that behavior. 
  * The deprecated linuxrc-based initial ramdisk (initrd) code has been [removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=996812c453ca>). The other initrd code is scheduled to follow in 2027, which will leave initramfs (which uses a filesystem in RAM instead of a disk image in RAM) the only supported way to boot the kernel. 

#### Filesystems and block I/O

  * Non-blocking updates to file modification times now [actually work](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=74554251dfc9>). Previously, they would return `-EAGAIN` unconditionally; now, that only happens when the filesystem would actually block. This makes non-blocking direct writes work on filesystems with fine-grained timestamps. 
  * Filesystems [no longer implement leases by default](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7e463614c97b>), and must instead opt-in. This resolves a number of problems caused by leases being available on filesystems that were never designed to handle them. Most popular filesystems do implement leases, but 9p and cephfs don't, for example. 
  * Historically, filesystems have reported errors in mutually incompatible ways. A [new set of helper functions](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=347b7042fb26>) makes it easier for filesystems to report errors to fsnotify in a consistent way. 
  * A new filesystem — "nullfs" — has [been added](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7416634fd6f1>) for use as the root filesystem of Linux systems. It's immutable and completely empty, containing no data whatsoever. This simplifies the boot process, because user space can mount other filesystems on top of it and then use the [ `pivot_root()`](<https://www.man7.org/linux/man-pages/man2/pivot_root.2.html>) system call to make those the new root, rather than having to clean up the contents of initramfs and re-use the root filesystem. 
  * In support of [ Checkpoint/Restore in Userspace](<https://criu.org/Main_Page>) (CRIU), the [`statmount()`](<https://man7.org/linux/man-pages/man2/statmount.2.html>) system call [ can now report information about the mount associated with a file descriptor](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d5bc4e31f2a3>). 
  * The EROFS maintainers have [enabled](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8f2fb72fd17e>) LZMA compression by default, and marked DEFLATE and Zstandard compression as no longer experimental. The filesystem also [shares page-cache entries](<https://lwn.net/Articles/1055062/>) for identical files on separate EROFS filesystems. 
  * Filesystems that need to calculate checksums or parity over data can [use bounce buffers](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c9d114846b38>) to store a copy of the data during direct I/O. See [this article](<https://lwn.net/Articles/1045006/>) for details. 
  * Btrfs [ now supports](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8912c2fd5830>) direct I/O when the block size exceeds the system's page size. 
  * XFS's [ autonomous self-healing support](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=04a65666a695>) has been merged; see [this article](<https://lwn.net/Articles/1055062>) for details. 

#### Hardware support

  * **GPIO and pin control** : ROHM bd72720 GPIO devices. 
  * **Graphics** : CSW MNE007QB3-1 panels, AUO B140HAN06.4 panels, AUO B140QAX01.H EDP panels, Sitronix ST7920 panels, Samsung LTL106HL02 panels, LG H546WF1-ED01 panels, HannStar HSD156J panels, BOE NV130WUM-T08 panels, Innolux G150XGE-L05 panels, Anbernic RG-DS panels, RK3368 HDMI controllers, RK3506 chips, Genio 510/700/1200-EVK HDMI outputs, and Radxa NIO-12L HDMI outputs. 
  * **Hardware monitoring** : MT8196 and MT7987 Mediatek heat sensors, RZ/T2H and RZ/N2H Renesas heat sensors, HiTRON HAC300S power supplies, Monolithic MP5926 hot-swap controllers, STEF48H28 hot-swap controllers, Pro WS TRX50-SAGE WIFI A and ROG MAXIMUS X HERO chips, Dell OptiPlex 7080 computers, F81968 I/O chips, ASUS Pro WS WRX90E-SAGE SE chips, SHT85 sensors, P3T1035 temperature sensors, and P3T2030 temperature sensors. 
  * **Media** : TI video input ports, os05b10, s5k3m5, and s5kjn1 camera sensors, and Synopsys CSI-2 receivers. 
  * **Miscellaneous** : Renesas RZ/V2N SoCs and Rock Band 4 PS4 and PS5 guitars, ATCSPI200 SPI devices, AXIADO AX300 SPI devices, NXP XPI SPI devices, and Renesas RZ/N1 SPI devices. 
  * **Networking** : Huawei hinic3 PF ethernet cards, Motorcomm YT6801 PCIe ethernet controllers, MaxLinear MxL862xx switches, RealTek RTL8127ATF 10G Fiber SFP NICs, RZ/G3L GBETH SoC NICs and QCC2072 WiFi chipsets. 
  * **Power** : Maxim MAX776750 PMICs, Realtek RT8902 level shifters, Samsung S2MPG11 PMICs, and Texas Instruments TPS65185 PMICs. 
  * **Sound** : NXP i.MX952 application processor, Realtek RT1320 and RT5575 audio codecs, and Sophogo CV1800B chips. 

#### Miscellaneous

  * The vDSO [now provides](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b205656daf93>) a 64-bit version of `clock_getres()`. 
  * With this version, the kernel [ supports](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8ea39d960c9f>) SPI devices with multiple data lanes that transmit in parallel. 

#### Security-related

  * SELinux can now [control BPF token access](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5473a722f782>). BPF tokens allow unprivileged processes to perform certain privileged BPF operations; see [this article](<https://lwn.net/Articles/935195/>) for details. 
  * The kernel [ supports](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=64edccea594c>) ML-DSA post-quantum signatures, and can [use them to authenticate kernel modules](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0ad9a71933e7>). 
  * The option to sign kernel modules with schemes involving SHA-1 hashes has [been removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=148519a06304>), although the kernel remains able to load modules signed this way, for now. 
  * `NETFILTER_PKT` audit records [now contain](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=15b0c43aa621>) the source and destination port numbers for inspection. 

#### Virtualization and containers

  * Container runtimes can use the new [ `OPEN_TREE_NAMESPACE`](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=157d3d6efd5a>) option to open a new mount namespace without cloning an existing mount namespace. This should make starting a new container faster on systems with many mounts. 

#### Internal kernel changes

  * A [reimplementation](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c27cea4416a3>) of RCU task traces has resulted in the [deprecation](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e55c2e287174>) of the `rcu_read_lock_trace()` and `rcu_read_unlock_trace()` functions. 
  * The kernel has added [ an official policy on tool-generated content](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a66437c27979577fe1feffba502b9eadff13af7d>). To encourage the tools themselves to follow it, there is also [ documentation aimed at LLMs](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit?id=78d979db6cef557c171d6059cbce06c3db89c7ee>). 
  * The `kmalloc_*()` family of functions (which allocate based on the required size) are [poised to be replaced](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bdc5071d7f7b>) with `kmalloc_obj_*()` functions (which allocate based on the provided type) during this release cycle. The new functions will both make object-length-calculation errors less common and provide for possible type-based hardening of the kernel. 
  * A [number of Rust changes](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a9aabb3b839a>) were made to use the recently-vendored `syn` crate to implement macros — changes which, ironically, actually reduced the amount of Rust code in the kernel by cleaning up the previous ad-hoc macro definitions. 
  * Support for Sparse context analysis (which helps find locking bugs, although not well) was [ removed](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5b63d0ae94cc>) in favor of [compiler-based context analysis](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8f32441d7a53>) in Clang 22. The compiler-based analysis should catch more locking bugs with fewer false positives; see [this article](<https://lwn.net/Articles/1012990/>) for details. 
  * The kernel's build configuration has [new syntactic sugar](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=76df6815dab7>): "depends on X if Y", standing in for "depends on X || !Y". 
  * Sheaf caches are [all cached per-CPU](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=815c8e35511d>), a change that has [been in the works](<https://lwn.net/Articles/1010667/>) for nearly a year. This change reduces the amount of cross-CPU contention caused by allocating new pages from the kernel's slab allocator. 
  * s390 machines now have [the same kinds of poison pointers](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0d453ba04044>) (which have hex value `0xdead000000000000` on s390) as other architectures, which allow the kernel to track DMA mappings from the networking page pool, among other things. 
  * The DRM subsystem has [ given up on integration](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=939faf71cf7c>) with the kernel debugger (kgdb) for now. The move is motivated by the difficulty of supporting kgdb on modern hardware. 
  * The new `__counted_by_ptr()` annotation [ marks members of a structure that specify the length of an object behind a pointer](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=150a04d817d8>), like `__counted_by()` does for arrays in a structure. 

The merge window is not quite half over, so as usual there will be a follow-up article once it closes, on February 22 if all goes as planned. For now, though, the 7.0 release is following the trend of recent Linux releases: packed with incremental improvements, and no huge changes. One thing that didn't make it into this release is [ support for revocable driver interfaces in C](<https://lwn.net/Articles/1058041/>); that patch set may just be pushed off to 7.1, or may face stiffer resistance.
