---
title: The first part of the 6.14 merge window
url: https://lwn.net/Articles/1005651/
date: "January 23, 2025"
category: "Releases-6.14"
author: "By Jonathan Corbet January 23, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
January 23, 2025

As of this writing, just over 4,300 non-merge changesets have been pulled into the mainline repository for the 6.14 release. Many of the pull requests this time around include remarks saying that activity has been relatively low this time around, presumably due to the holidays. So those 4,300 changesets are probably closer to the merge-window halfway point than usual. Much of the work merged thus far looks more like incremental improvements than major new initiatives, but there still have been a number of interesting changes in the mix. 

Some of the most significant changes pulled into the mainline so far are: 

#### Architecture-specific

  * The PowerPC architecture has gained [lazy preemption](<https://lwn.net/Articles/994322/>) support. 
  * X86 systems using AMD's [Secure Encrypted Virtualization](<https://www.amd.com/en/developer/sev.html>) feature now support a secure timestamp counter for guests. In short, it allows guests to read timestamps that cannot be manipulated by the host. 
  * AMD's energy-use counters for CPU cores are now supported in the perf events subsystem. 

#### Core kernel

  * The `pid_max` sysctl knob sets the highest number that can be used for a process ID; it has the effects of limiting the size of PID values and of limiting the total number of processes that may exist. In 6.14, `pid_max` is now tied to the PID namespace, allowing it to be set independently within containers. It is hierarchical, so no namespace can set `pid_max` to a value higher than that found in any of its parent namespaces. See [this commit](<https://git.kernel.org/linus/7863dcc72d0f>) for more information about this change. 
  * When a program is launched with [`execveat()`](<https://man7.org/linux/man-pages/man2/execveat.2.html>), the name of the executed file as stored in its directory entry will be shown in `/proc` rather than (as is done in current kernels) the file-descriptor number that was used. See [this article](<https://lwn.net/Articles/999770/>) for details on this change. 
  * The new "dmem" control-group controller regulates access to device memory, such as that found on graphics cards. Documentation is sparse, but there is [a brief guide](<https://docs.kernel.org/next/admin-guide/cgroup-v2.html#dmem>) to the configuration of this controller available. 

#### Filesystems and block I/O

  * The pidfdfs filesystem can now create file handles (when requested by a [`name_to_handle_at()`](<https://man7.org/linux/man-pages/man2/open_by_handle_at.2.html>) call); these can be used to create a system-wide unique identifier for processes even on 32-bit systems. It is also now possible to bind-mount pidfds. 
  * The [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) system call can now return the required alignment for read operations on a file; that alignment may be different than the requirement for writes, and some applications can benefit from knowing both. 
  * Some Btrfs configurations give the filesystem a choice of multiple devices when the time comes to read a specific block. In current kernels, the PID of the reading process is used to make that decision, but that will focus all read traffic onto a single device in a single-reader workload. The 6.14 kernel adds a couple of new policy options that can implement either round-robin read balancing or simply focus reads onto a specific device. See [this commit](<https://git.kernel.org/linus/6d7a9154955e>) for instructions on enabling round-robin, or [this one](<https://git.kernel.org/linus/c86aae73bd58>) to set a specific device. 
  * The bcachefs filesystem has a lot of changes after missing the 6.13 development cycle; these include a major on-disk format change that will require a ""big and expensive"" format upgrade. These changes include self-healing improvements, filesystem-checking time ""improved by multiple orders of magnitude"", and more; see [this merge message](<https://git.kernel.org/linus/2622f2904170>) for more information. 
  * The md-linear device-mapper target (which essentially concatenates block devices) was removed in 6.8 as being deprecated and unmaintained. It seems that there were still users of this target, though, so it [has been restored](<https://git.kernel.org/linus/127186cfb184>) for 6.14. This change is also marked for the stable updates, so should propagate to the older kernels as well. 

#### Hardware support

  * **Clock** : Qualcomm X1P42100 graphics clock controllers, Qualcomm QCS615 and SM8750 global clock controllers, Qualcomm SM8750 TCSR clock controllers, Qualcomm SM8750 display clock controllers, Qualcomm IPQ CMN PLL clock controllers, and Qualcomm SM6115 low power audio subsystem clock controllers. 
  * **Graphics** : Synopsys Designware MIPI DSI host DRM bridges and ZynqMP DisplayPort audio interfaces. 
  * **Hardware monitoring** : TI TPS25990 monitoring interfaces, Intel common redundant power supply monitors, and Analog Devices ADM1273 hot-swap controllers. 
  * **Miscellaneous** : NVMe PCI endpoint function targets, Loongson memory controllers, AMD AI engines, STMicroelectronics LED1202 I2C LED controllers, TI LP8864/LP8866 4/6 channel LED drivers, KEBA SPI interfaces, and Airoha EN7581 SoC CPU-frequency controllers. 
  * **Networking** : NXP S32G/S32R Ethernet interfaces, Realtek 8922AE-VS PCI wireless network adapters, and QNAP microcontroller unit cores. 

#### Miscellaneous

  * The `samples` directory in the kernel repository contains a new program, `mountinfo`, which demonstrates the use of the [`statmount()` and `listmount()`](<https://lwn.net/Articles/950569/>) system calls. 
  * When Rust 1.84.0 (or later) is available, Rust code in the kernel will use the [`derive(CoercePointee)`](<https://github.com/rust-lang/rust/issues/123430>) feature for pointer coercion. That feature is on the Rust-language stabilization track, and its use is an important step toward using only stable Rust features in the kernel. [This merge message](<https://git.kernel.org/linus/e3610441d1fb>) shows how it can be used. 

#### Networking

  * The [RxRPC](<https://docs.kernel.org/networking/rxrpc.html>) protocol implementation can now make use of huge UDP frames for better throughput. Support for the [RACK-TLP](<https://datatracker.ietf.org/doc/html/rfc8985>) loss-detection algorithm has also been added. 
  * There is a new per-network-namespace sysctl knob — `tcp_tw_reuse_delay` — that controls how long the system will wait before reusing the port number of a closed TCP socket; its value is in milliseconds. 
  * It is now possible to select whether an interface MAC or PHY should be used as the provider of PTP timestamps; [this merge message](<https://git.kernel.org/linus/bc6a5efe3dcd>) gives some examples of how to do this that are presumably intelligible to people familiar with such things. 
  * IPsec IP-TFS/AGGFRAG ([RFC 9347](<https://datatracker.ietf.org/doc/rfc9347/>)) is now supported. 

#### Security-related

  * The "xperms" SELinux feature allows policies to target specific `ioctl()` calls or netlink messages. In-kernel documentation is missing, but [this wiki page](<https://selinuxproject.org/page/XpermRules>) has some information. 

#### Internal kernel changes

  * The kernel's annotation system, used to add information about code (such as "this jump is safe without a retpoline") would previously create a different ELF section for each annotation type. There is now [a generic annotation infrastructure](<https://git.kernel.org/linus/2116b349e29a>) that gathers all of that information into a single section. 

The 6.14 merge window can be expected to remain open through February 2, with the 6.14 release most likely happening on March 23\. This timing seems more certain than usual, just because it will maximize editorial pain at LWN due to the [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) starting on March 24\. One way or another, we'll survive the experience and tell you how it goes.
