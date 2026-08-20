---
title: The second half of the 6.16 merge window
url: https://lwn.net/Articles/1023075/
date: "June 9, 2025"
category: "Releases-6.16"
author: "By Daroc Alden June 9, 2025"
---

By **Daroc Alden**  
June 9, 2025

The 6.16 merge window [ closed](<https://lwn.net/ml/all/CAHk-%3DwiiqYoM_Qdr3L15BqJUyRi6JjR02HSovwwz%2BBXy7DdVeA%40mail.gmail.com/>) on June 8, as expected, containing 12,899 non-merge commits. This is slightly more than the 6.15 merge window, but well in line with expectations. 7,353 of those were merged after [the summary of the first half of the merge window](<https://lwn.net/Articles/1022512>) was written. More detailed statistics can be found in [the LWN kernel source database](<https://lwn.net/ksdb/>). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

As usual, the second half of the merge window contained more bug fixes than new features, but there were many interesting features that made their way in as well: 

#### Architecture-specific

  * The [ `getrandom()`](<https://www.man7.org/linux/man-pages/man2/getrandom.2.html>) system call is now [much faster](<https://git.kernel.org/linus/89079520cef6>) on RISC-V. It is now handled entirely within the vDSO. 
  * RISC-V kernels now support new vendor extensions from [SiFive](<https://git.kernel.org/linus/1a3f6980889d>), as well as the [Zicbop](<https://git.kernel.org/linus/847689d2a0c4>), [Zabha](<https://git.kernel.org/linus/415a8c81da3d>), and [Svinval](<https://git.kernel.org/linus/a56972698810>) extensions. They also include [the supervisor binary interface (SBI) firwmare features (FWFT) extension](<https://git.kernel.org/linus/51f1b16367df>), which is needed for SBI 3.0, the latest version. 
  * LoongArch [now supports](<https://git.kernel.org/linus/9559d5806319>) up to 2048 CPUs, the maximum that the architecture can handle. The architecture now also has [multi-core scheduling](<https://git.kernel.org/linus/93f437315660>). 

#### Core kernel

  * Unix-domain sockets can be used to transfer file descriptors; it is now [possible for a program to opt-out](<https://lwn.net/Articles/1023085/>) of that ability, which may be important to preventing denial-of-service attacks. 
  * The ring buffer used for kernel tracing can now be [ mapped into memory in user space](<https://git.kernel.org/linus/c2a08311427cc8c5c547e5d700cb2f93d63fcb2a>). 
  * [A new API](<https://git.kernel.org/linus/fc33e4b44b27>) will let virtual memory allocations [persist](<https://lwn.net/Articles/1015997>) across [kexec handovers](<https://docs.kernel.org/next/admin-guide/mm/kho.html>). 
  * Crash-dump kernels (the special kernel that runs after a kernel crash to produce a report) can now [reuse existing LUKS keys](<https://git.kernel.org/linus/180cf31af7c3>). This lets crash dumps be made to encrypted filesystems, which was not previously possible. 
  * The kernel memory accounting done by the memory control-group code can now be [performed](<https://git.kernel.org/linus/25352d2f2dc6>) in a non-maskable interrupt (NMI) context. This is important because BPF programs can run in NMI contexts, and may need to allocate memory in the kernel, which in turn needs to be accounted for. 
  * [ NUMA weighted interleaving](<https://lwn.net/Articles/1016842>) is now [ automatically tuned](<https://git.kernel.org/linus/e341f9c3c8412e57fe0042a33a2640245ecdf619>), providing better utilization of memory bandwidth in systems with data striped across multiple NUMA nodes. 

#### Filesystems and block I/O

  * [OrangeFS](<https://www.orangefs.org/>) now [makes use of](<https://git.kernel.org/linus/b04f9f88936c>) the [ new mount API](<https://lwn.net/Articles/759499/>), as does [UFS](<https://git.kernel.org/linus/edb94482e9d6>). 
  * The limit on read and write sizes for NFS filesystems has [been raised to 4MB](<https://git.kernel.org/linus/2c26b68cd5c5>). ""The default remains at 1MB, but risk-seeking administrators now have the ability to try larger I/O sizes with NFS clients that support them."" 
  * Users with the `CAP_SYS_ADMIN` capability in a user namespace (and no privileges in the root namespace) [can now watch filesystems and mounts with fanotify](<https://git.kernel.org/linus/58f5fbeb367f>). 
  * The ext2 filesystem has [deprecated support for DAX](<https://git.kernel.org/linus/d5a2693f93e4>), since it isn't widely used. The ext2 filesystem itself isn't widely used either, but it does serve as a stable reference implementation of a filesystem. Since persistent memory has not become as widely used as once expected, supporting it in a reference implementation doesn't make much sense. DAX support in ext2 is expected to be completely removed at the end of 2025. 
  * FUSE filesystems [can now invalidate](<https://git.kernel.org/linus/2396356a945b>) all existing cached directory entries (dentries) in a single operation. 
  * The overlayfs filesystem [now supports](<https://git.kernel.org/linus/b71db54ef3b8>) data-only layers with dm-verity in user namespaces. This allows trusted metadata layers to be combined with untrusted data layers in unprivileged namespaces. 

#### Hardware support

  * **Clock** : SpacemiT K1 SoCs, Sophgo SG2044 SoCs, T-HEAD TH1520 video-output clocks, Qualcomm QCS8300 camera clocks, Allwinner H616 display-engine clocks, Samsung ExynosAutov920 CPU cluster clock controllers, Renesas RZ/V2N R9A09G056 SoCs, Sophgo CV1800 clocks, and NXP S32G2/S32G3 clocks. 
  * **GPIO and pin control** : Mediatek MT6893 and MT8196 SoCs, Renesas RZ/V2N SoCs, MediaTek Dimensity 1200 (MT6893) I2C, Sophgo SG2044 I2Ci, Renesas RZ/V2N R9A09G056 I2C, Rockchip RK3528 I2C, and NXP Freescale i.MX943 SoCs. 
  * **Graphics** : Amlogic C3 image-signal processors. 
  * **Hardware monitoring** : Dasharo fans and temperature sensors, KEBA fan controllers and battery monitoring controllers, MAX77705 ICs, MAXIMUS VI HERO and ROG MAXIMUS Z90 Formula motherboards, SQ52206 energy monitors, lt3074 linear regulators, ADPM12160 DC/DC power modules, and MPM82504 and MPM3695 DC/DC power modules. 
  * **Industrial I/O** : DFRobot SEN0322 oxygen sensors. 
  * **Input** : ByoWave Proteus game controllers and Apple Magic Mouse 2s. 
  * **Media** : ST VD55G1 and VD56G3 image sensors and OmniVision OV02C10 image sensors. 
  * **Miscellaneous** : FSL vf610-pit periodic-interrupt timers, SGX vz89te integrated sensors, Maxim max30208 temperature sensors, TI lp8864 automotive displays, MT6893 MM IOMMUs, Sophgo CV1800 and SG2044 SoCs, Qualcomm sm8750 SoCs, Amlogic c3 and s4 SoCs, and Renesas RZ/V2H(P) R9A09G057 DMA controllers. 
  * **Networking** : Renesas RZ/V2H(P) SoC, Broadcom asp-v3.0 ethernet devices, AMD Renoir ethernet devices, RealTek MT9888 2.5G ethernet PHYs, Aeonsemi 10G C45 PHYs, Qualcomm IPQ5424 qusb2 PHYs, IPQ5018 uniphy-pcie devices, Mediatek MT7988 xs-PHYs, and Renesas RZ/V2H(P) usb2 PHYs. 
  * **Sound** : Fairphone FP5 sound card. 

#### Miscellaneous

  * Support for the STA2x11 video input port driver has [finally gone away](<https://git.kernel.org/linus/29d69273fefd>). 
  * The documentation generation script `scripts/bpf_doc.py` [ can now produce JSON output](<https://git.kernel.org/linus/cb4a11925268b13ebcac322775d78bdd4e1b26d3>) about BPF helpers and other elements of the BPF API. This change makes it easier for external tools to keep their knowledge of the BPF interface up to date. 
  * Writing "default" to the sysfs trigger of an LED device will [ now reset the trigger](<https://git.kernel.org/linus/06d99fcf1f87>) to that device's default. 
  * Compute express link (CXL) devices [now support](<https://git.kernel.org/linus/9f153b7fb5ae>) the reliability, availability, and serviceability (RAS) extensions. Most importantly, these let CXL devices participate in various error detection and correction schemes. 
  * This release includes a [number of improvements](<https://git.kernel.org/linus/0939bd2fcf33>) to `perf`, including [support for calculating system call statistics in BPF](<https://git.kernel.org/linus/1bec43f5239d>), better [demangling of Rust symbols](<https://git.kernel.org/linus/4d728bb93bab>), [more granular options](<https://git.kernel.org/linus/43a644699838>) for collecting memory statistics, a flag to [deliberately introduce lock contention](<https://git.kernel.org/linus/c42e219942cb>), and several more. 
  * USB audio devices now support audio offloading. This lets, for example, audio from a USB device to continue to flow even when the rest of the system is sleeping. In the [pull request](<https://git.kernel.org/linus/c0c9379f235d>), Greg Kroah-Hartman said: ""I think this takes the record for the most number of patch series (30+) over the longest period of time (2+ years) to get merged properly."" 

#### Networking

  * The contents of device memory [ can now be sent via TCP](<https://git.kernel.org/linus/1b98f357dadd>), allowing [ zero-copy transmission from a GPU to the wire](<https://lwn.net/Articles/979549/>). 
  * BPF can be used to implement traffic-control queueing disciplines (qdiscs) with a struct_ops program. 
  * Support for the [ datagram congestion control protocol](<https://en.wikipedia.org/wiki/Datagram_Congestion_Control_Protocol>) (DCCP) is being [removed](<https://git.kernel.org/linus/2a63dd0edf38>) following a long deprecation and no signs of having any users. DCCP was intended to prevent problems with UDP's lack of rate control, which have largely failed to materialize. It was originally [added](<https://lwn.net/Articles/149756>) in 2005. The hope is that this removal will enable cleanup of the parts of the TCP stack that are currently shared with DCCP. 
  * The kernel now supports using [ generic security services application programming interface](<https://en.wikipedia.org/wiki/Generic_Security_Services_Application_Programming_Interface>) (GSSAPI) [ for the AFS filesystem](<https://git.kernel.org/linus/5b38e821b929c23a3b7bfa2705cc7b0e76a3ee7b>), allowing connections to manage the encryption of connections to YFS and OpenAFS servers. 
  * OpenVPN now has [ a virtual driver](<https://git.kernel.org/linus/a8ae8a0e848e3506c95e45e7cb6e640502495f1a>) for offloading some operations to the kernel, which should make it faster, especially for large transfers. 

#### Security-related

  * The [ randstruct GCC plugin](<https://lwn.net/Articles/722293>), which makes it harder for attackers to access kernel data structures by randomizing their layout, is now [working again](<https://git.kernel.org/linus/f39f18f3c3531aa802b58a20d39d96e82eb96c14>), and has tests to keep it that way. The ARM_SSP_PER_TASK GCC plugin, which lets different tasks use different stack canaries, has [ been retired](<https://git.kernel.org/linus/b8e147973eca7e07fa0845350d77c9970263fcd7>), since its functionality is available in upstream GCC. 
  * Integrity Measurement Architecture (IMA) measurements can now be [carried across kexec invocations](<https://git.kernel.org/linus/7af6e3febb91>). A new kernel-configuration option, `IMA_KEXEC_EXTRA_MEMORY_KB` determines how much memory is set aside for new IMA measurements on a soft reboot. 
  * The measurements made by the trusted security manager (TSM; part of Intel's trust domain extensions, also known as TDX) are now [exposed as part of sysfs](<https://git.kernel.org/linus/4d2a7bfad5b7>). This gives user space the opportunity to make decisions based on attestations from the hardware. 
  * The performance overhead of SELinux has been [reduced](<https://git.kernel.org/linus/b5628b81bd19>) by adding a cache for directory-access decisions and support for wildcards in [ genfscon](<https://web.archive.org/web/20240723114331/http://arctic.selinuxproject.org/page/FileStatements>) policy statements. 
  * The kernel's EFI code has been [extended](<https://git.kernel.org/linus/0f9a1739dd0e>) to allow emitting a `.sbat` section with UEFI SecureBoot revocation information; the upstream kernel project won't maintain the revocation information, but individual distributions now have the access they need to the be able to ship their own revocation databases. 
  * The `.static_call_sites` section in loadable modules [is now made read-only](<https://git.kernel.org/linus/60b57b9cb002>) after module initialization. 

#### Virtualization and containers

  * 64-Bit Arm now supports [ transparent huge pages](<https://git.kernel.org/linus/a90e0017541d>) on non-protected guests when protected KVM is enabled. 
  * Nested virtualization support on 64-bit Arm is also working, although it remains disabled by default. 
  * x86 virtual machine hosts on KVM now [support TDX](<https://git.kernel.org/linus/0d20742b8e6b>), enabling the use of confidential guests on Intel processors. This change ""has been in the works for literally years"", and includes a large number of patches. 
  * KVM support on RISC-V is [no longer experimental](<https://git.kernel.org/linus/1f7c9d52b12ded6c99b5623d1e81bba7bb76c2f4>). 

#### Internal kernel changes

  * The power-management subsystem has [gained](<https://git.kernel.org/linus/0c905cadf38b>) Rust abstractions for managing CPU frequency, operating performance points (OPPs), and related power-management APIs. 
  * The kernel's minimum supported GCC version has been [updated](<https://git.kernel.org/linus/dee264c16a63>) to GCC 8 for all architectures; the update allows for two of the five remaining GCC plugins used in kernel builds to be removed. The corresponding minimum version of binutils is 2.30. 
  * A [bevy of memory-management changes](<https://git.kernel.org/linus/00c010e130e5>) includes more folio conversions, [Rust abstractions](<https://git.kernel.org/linus/5bb9ed6cdfeb>) for core memory-management operations, better support for memory compaction, and [the removal of `VM_PAT`](<https://git.kernel.org/linus/ed1a7814036c>). 
  * Rust test error messages are now more tightly integrated into KUnit when using [assertions](<https://git.kernel.org/linus/36174d16f3ec>) and [results](<https://git.kernel.org/linus/950b306c296e>). Rust code can now also [make use of XArrays](<https://git.kernel.org/linus/06ff274f25e9>). 

The 6.16 kernel now goes into the stabilization period, with the final release expected July 27 or August 3.
