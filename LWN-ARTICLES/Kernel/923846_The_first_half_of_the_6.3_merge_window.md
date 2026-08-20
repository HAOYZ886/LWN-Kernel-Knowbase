---
title: The first half of the 6.3 merge window
url: https://lwn.net/Articles/923846/
date: "February 23, 2023"
category: "Releases-6.3"
author: "By Jonathan Corbet February 23, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
February 23, 2023

As of this writing, 5,776 non-merge changesets have been pulled into the mainline kernel for the 6.3 release; that is a bit less than half of the work that was waiting in linux-next before the merge window opened. This merge window is thus well underway, but far from complete. Quite a bit of significant work has been pulled so far; read on to see what entered the kernel in the first half of the 6.3 merge window. 

Changes merged to date include: 

#### Architecture-specific

  * A large set of old and unused Arm board files [has been removed](<https://git.kernel.org/linus/ff0c7e18629b>), reducing the size of the kernel tree by over 150,000 lines. [This (6.0) commit](<https://git.kernel.org/linus/7d0d3fa7339e>) describes the list of systems for which board files have been removed. Meanwhile, devicetree files have been added to support [46 new arm64 systems](<https://git.kernel.org/linus/950b6662e26e>). 
  * The new `virtconfig` build target for arm64 systems creates a relatively lightweight configuration intended to be booted on virtual systems. 
  * AMD's "automatic IBRS" feature is now supported. This is a Spectre defense that restricts indirect-branch speculation with less of a performance cost than that imposed by retpolines. 
  * The m68k architecture has gained support for system-call filtering with [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>). 
  * Arm [scalable matrix extension 2](<https://docs.kernel.org/arm64/sme.html>) instructions are now supported. 
  * BPF trampolines are now fully supported on s390x and RISC-V RV64 systems. 

#### Core kernel

  * The list of enhancements to the kernel's embryonic support for the Rust language is relatively small this time, but that support is, [according to Miguel Ojeda](<https://lwn.net/ml/linux-kernel/20230212183249.162376-1-ojeda@kernel.org/>), ""getting closer to a point where the first Rust modules can be upstreamed"". These changes include the removal of a non-applicable part of the alloc crate, an [implementation](<https://git.kernel.org/linus/9dc043655003>) of the [Arc](<https://doc.rust-lang.org/std/sync/struct.Arc.html>) type (which provides a reference-counted pointer), the [ScopeGuard](<https://git.kernel.org/linus/4d4692a2ff83>) type (which runs some cleanup code when it goes out of scope), and the [ForeignOwnable](<https://git.kernel.org/linus/0fc4424d24a2>) type, which facilitates moving pointers between Rust and C code. 
  * There is [a new document](<https://git.kernel.org/linus/16c294a6aad8>) covering the stability expectations for BPF kfuncs; it describes the current status in [the ongoing discussion](<https://lwn.net/Articles/921088/>) of how stable the BPF API should be. 
  * The `cgroup.memory=nobpf` command-line parameter disables memory accounting for BPF programs; see [this merge message](<https://git.kernel.org/linus/ab86cf337a5b>) for a discussion of the motivation behind this feature. 
  * There is a new [red-black tree](<https://lwn.net/Articles/500355/>) data structure available to BPF programs. See [this merge message](<https://git.kernel.org/linus/c8ea09974f33>) for more information. 
  * The [restartable sequences](<https://lwn.net/Articles/883104/>) mechanism now exports a "per-memory-map concurrency ID" to processes. This ID can be thought of (and treated like) a CPU number, but the numbers are kept as close to zero as possible. Its purpose is to enable more efficient per-CPU data structures in applications that are only using a subset of the CPUs on a large system. [This commit](<https://git.kernel.org/linus/af7f588d8f73>) contains some more information. 

#### Filesystems and block I/O

  * The tmpfs filesystem now supports [ID-mapped mounts](<https://lwn.net/Articles/896255/>). 
  * Erofs has gained support for per-CPU file-data decompression, leading to reduced data-access latency. 
  * The Btrfs block allocator will now segregate extents by their size, so that any given block group is limited to extents that are small (less than 128KB), medium (up to 8MB), or large. This evidently reduces fragmentation, especially in workloads where allocation size correlates with file lifetime — something that evidently actually happens. See [this commit message](<https://git.kernel.org/linus/52bb7a2166af>) for some details. 
  * Rotating disk drives still exist, and are even becoming more complex: multi-actuator drives have independently controllable arms that, for best performance, must all be kept busy. The BFQ I/O scheduler has gained support for such drives; [this commit message](<https://git.kernel.org/linus/9778369a2d6c>) has a bit more information on how it works. 

#### Hardware support

  * **GPIO and pin control** : Qualcomm QDU1000/QRU1000, IPQ5332, SA8775P, and SM8550 pin controllers, Mediatek MT7981 pin controllers, and StarFive JH7110 pin and GPIO controllers. 
  * **Hardware monitoring** : MPS MPQ7932 regulators, HPE GXP fan controllers, NXP MC34VR500 power-management ICs, and Infineon TDA38640 voltage regulators. 
  * **Input** : EVision keyboards and Steam Deck force feedback controllers. 
  * **Miscellaneous** : Xilinx ZynqMP on-chip-memory controllers, MediaTek low-voltage thermal sensor controllers, Intel topology aware register/pm capsule interfaces, Aspeed ACRY RSA engines, StarFive JH7110 random number generators, Maxim MAX20411 single step-down converters, and Broadcom BCMBCA HS SPI controllers. 
  * **Networking** : Microchip KSZ9563/LAN937x Ethernet switch PTP clocks, Realtek RTL8188EU wireless interfaces, Ocelot VSC7511, VSC7512, VSC7513 and VSC7514 external switches, Amlogic GXL-based MDIO bus multiplexers, Motorcomm 8531 PHYs, and Qualcomm WiFi 7 (ath12k) interfaces. 
  * **Sound** : MediaTek MT8188 controllers, Iron Device SMA1303 audio amplifiers, Renesas IDT821034 quad PCM codecs, Awinic AW88395 audio amplifiers, Realtek RT712 SDCA codecs, and Infineon PEB2466 quad PCM codecs. 
  * **Also** : preliminary support for writing [human-interface device drivers in BPF](<https://lwn.net/Articles/909109/>) has been merged, though the mechanism for distributing such drivers is still to be worked out. See [this document](<https://docs.kernel.org/next/hid/hid-bpf.html>) for more information. 

#### Networking

  * Support for the Physical Layer Collision Avoidance (PLCA) Reconciliation Sublayer has been added; it is said to improve access performance on shared media Ethernet. [This documentation patch](<https://git.kernel.org/linus/8580e16c28f3>) describes how to configure and use this feature. 
  * The "wireless extensions" API for the control of WiFi interfaces [ran into trouble](<https://lwn.net/Articles/202838/>) in 2006, but is still supported as an emulation layer. This API will no longer be supported for WiFi 7 (802.11be) interfaces, since it is unable to configure all of the available features. The use of the wireless extensions API will generate a warning for most current devices as of 6.3. 
  * The process of documenting the netlink API continues; the results can be seen in the [core API](<https://docs.kernel.org/next/core-api/netlink.html>) and [user-space API](<https://www.kernel.org/doc/html/next/userspace-api/netlink/>) manuals. Also added is [a new tool](<https://git.kernel.org/linus/be5bea1cc0bf>) to generate netlink protocol code from YAML specifications. 
  * The new `IP_LOCAL_PORT_RANGE` socket option makes it easier for multiple hosts to make outgoing connections through a NAT gateway; [this commit](<https://git.kernel.org/linus/91d0b78c5177>) contains details. 
  * Multi-path TCP can now handle mixed flows using both the IPv4 and IPv6 protocols. 
  * [BIG TCP](<https://lwn.net/Articles/884104/>) support has been extended to IPv4. 
  * The new `default_rps_mask` sysctl knob allows the creation of a default, per-net-namespace [receive packet steering (RPS)](<https://www.kernel.org/doc/html/latest/networking/scaling.html?highlight=rps#rps-receive-packet-steering>) configuration. 
  * Support for a number of queuing disciplines (specifically class-based queuing (CBQ), ATM virtual circuits (ATM), differentiated service marker (dsmark), traffic-control index (tcindex), and resource reservation protocol (RSVP)) has been removed due to a lack of maintenance and interest. 

#### Internal kernel changes

  * The old memory-allocation function `get_kernel_pages()` has been removed now that there are no more in-tree users. 

The 6.3 merge window can be expected to remain open until March 5, at which point 6.3-rc1 will come out and the kernel will enter the stabilization phase of the development cycle. Quite a few more changes are poised to enter the mainline before that happens, though; tune in once the merge window closes for a summary of the rest of that work.
