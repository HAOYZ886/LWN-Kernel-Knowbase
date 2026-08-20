---
title: The rest of the 6.17 merge window
url: https://lwn.net/Articles/1032095/
date: "August 11, 2025"
category: "Releases-6.17"
author: "By Jonathan Corbet August 11, 2025"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 11, 2025

The [6.17-rc1](<https://lwn.net/Articles/1033167/>) prepatch was released by Linus Torvalds on August 10; the 6.17 merge window is now closed. There were 11,404 non-merge changesets pulled into the mainline this time around, a little over 7,000 of which came in after [the first-half merge-window summary](<https://lwn.net/Articles/1031713/>) was written. As one would expect, quite a few changes and new features were included in that work. 

Some of the most significant changes pulled into the mainline during the second half of the 6.17 merge window are: 

#### Architecture-specific

  * Support for BPF has been improved for the LoongArch architecture, which can now handle dynamic code modification, BPF trampolines, and [struct ops](<https://lwn.net/Articles/1032199/>) programs. 
  * S390 systems now support the swapping and migration of transparent huge pages. 

#### Core kernel

  * The BPF subsystem now exports a set of standard (but read-only) string operations for BPF programs; there is no documentation, but the functions can be found in [this commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e91370550f1f>). 
  * BPF programs now have standard output and error streams that can be used to communicate back to user space; see [this commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5ab154f1463a>) for details. 
  * The new DAMON_STAT kernel module provides simplified monitoring of memory-management activity in the system; see [this changelog](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=369c415e6073>) and [`Documentation/admin-guide/mm/damon/stat.rst`](<https://docs.kernel.org/next/admin-guide/mm/damon/stat.html>) for more information. 
  * It is now possible to control the aggressiveness of the proactive-reclaim machinery on a per-NUMA-node basis, allowing some nodes to more actively evict pages than others. See [this changelog](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b980077899ea>) for details. 
  * The [extensible scheduler class](<https://lwn.net/Articles/922405/>) now has support for control-group-based bandwidth control (specifically the `cpu.max` parameter described in [this document](<https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html#cpu-interface-files>)). 

#### Hardware support

  * **Clock** : Renesas RZ/T2H clocks, SpacemiT reset controllers, Qualcomm SM6350 video clock controllers, Qualcomm SC8180X camera clock controllers, multiple Qualcomm QCS615 clock controllers, and multiple Qualcomm Milos clock controllers. 
  * **GPIO and pin control** : ESWIN EIC7700 pin-control units, Qualcomm Milos pin controllers, STMicroelectronics STM32 hardware debug port pin controllers, and MediaTek MT8189 pin controllers. 
  * **Graphics** : Renesas R69328 720x1280 DSI video mode panels, Himax HX83112B-based DSI panels, and Intel Discrete Graphics non-volatile memory. 
  * **Miscellaneous** : Qualcomm M31 eUSB2 PHYs, Sophgo CV1800/SG2000 series SoC DMA multiplexers, Sophgo DesignWare PCIe controllers (host mode), Renesas I3C controllers, Broadcom BCM74110 mailboxes, and Aspeed AST2700 mailboxes. 
  * **Networking** : Qualcomm IPQ5018 Internal PHYs, Airoha AN7583 MDIO bus controllers, Broadcom 50/100/200/400/800 gigabit Ethernet cards, Microchip Azurite DPLL/PTP/SyncE devices, and Realtek 8851BU and 8852BU USB wireless network (Wi-Fi 6) adapters. 

#### Miscellaneous

  * The [runtime verification subsystem](<https://docs.kernel.org/trace/rv/index.html>) has gained support for [linear temporal logic monitors](<https://lwn.net/Articles/1030685/>); [this commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e93648e86273>) provides documentation. There is a new monitor, `rtapp`, that looks for common problems in realtime applications; see [this commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=670ff946b9bd>) for documentation. Finally, the new `nrp`, `sssw`, and `opid` monitors are available for internal scheduler testing. 
  * The automatic mounting of the tracefs virtual filesystem on `/sys/kernel/debug/tracing` has been deprecated; scripts should be using `/sys/kernel/tracing` instead. The current plan is to remove that automatic mount in 2030. 
  * See [this merge message](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f4f346c34659>) for a summary of the numerous changes to the `perf` tool in 6.17. 
  * There is a new option to reserve space for kernel crash dumps from the [contiguous memory allocator](<https://lwn.net/Articles/486301/>), making that memory available for use by the kernel prior to a crash. See [this documentation patch](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ce1bf19a34df>) for details. 

#### Networking

  * Support for [RFC 6675 loss detection](<https://www.rfc-editor.org/rfc/rfc6675>) has been removed from the kernel. This protocol has long been considered obsolete and has not been used by default since 2018. At this point, it seems that everybody is using [RACK-TLP](<https://datatracker.ietf.org/doc/html/rfc8985>) instead. 
  * The [power sourcing equipment (PSE)](<https://docs.kernel.org/next/networking/pse-pd/index.html>) implementation has gained support for configurable budget-evaluation strategies, which are ""utilized by PSE controllers to determine which ports to turn off first in scenarios such as power budget exceedance"". See [this changelog](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ffef61d6d273>) for some more information. 
  * Support for gateway routing has been added to the [Management Component Transport Protocol (MCTP)](<https://docs.kernel.org/networking/mctp.html>) subsystem; see [this merge message](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d23647fd547b>) for an overview. 
  * The new `SO_INC` option for `AF_UNIX` sockets mirrors `TCP_INQ`; it will cause a control message to be placed on the socket indicating how much data is available to be read there. Similarly, `SIOCINQ` has been added for the [VSOCK address family](<https://man7.org/linux/man-pages/man7/vsock.7.html>). 
  * The TCP implementation has traditionally been forgiving about accepting data beyond the advertised receive window; that comes to an end in 6.17, which enforces the window limit more strictly. 
  * Multipath TCP now supports the `TCP_MAXSEG` socket option to set the maximum size of outgoing segments. 
  * Support for the [DualPI2 (RFC 9332)](<https://datatracker.ietf.org/doc/rfc9332/>) congestion-control protocol has been added; see [this commit](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=8f9516daedd6>) for an overview of the Linux implementation. 
  * The new `force_forwarding` sysctl knob allows the administrator to enable forwarding on specific IPv6 interfaces. 

#### Security-related

  * The [AppArmor](<https://apparmor.net/>) security module has gained the ability to control access to `AF_UNIX` sockets. See [this commit changelog](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c05e705812d1>) for an overview of how it works. 

#### Virtualization and containers

  * KVM on Arm systems now supports the [GICv5 interrupt controller](<https://community.arm.com/arm-community-blogs/b/architectures-and-processors-blog/posts/introducing-gicv5>). 

#### Internal kernel changes

  * Memory managed by the networking subsystem's page pool is now referred to using [`struct netmem_desc`](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f3d85c9ee510>) rather than `struct page`. This work is part of the ongoing process of moving from `struct page` to descriptors specific to the way each folio is being used. 
  * The deferred unwind infrastructure — a needed precursor for the work to [add SFrame-based stack unwinding](<https://lwn.net/Articles/1029189/>) — has been merged. The SFrame work itself will seemingly wait for another development cycle. 
  * Rust support has been added for the `warn_on!()` macro, delayed workqueue items, a `UserPtr` type for user-space pointers, and more; see [this merge message](<https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=352af6a011d5>) for a list. 
  * The `gconfig` kernel-configuration tool has been migrated to the GTK 3 toolkit. 
  * There were 171 exported symbols removed, and 523 added this time around; see [this page](<https://lwn.net/Articles/1033175/>) for the full list. Developers also removed seven kfuncs (`scx_bpf_consume()`, `scx_bpf_dispatch()`, `scx_bpf_dispatch_from_dsq()`, `scx_bpf_dispatch_from_dsq_set_slice()`, `scx_bpf_dispatch_from_dsq_set_vtime()`, `scx_bpf_dispatch_vtime()`, and `scx_bpf_dispatch_vtime_from_dsq()`) and added 15 others (`bpf_arena_reserve_pages()`, `bpf_cgroup_read_xattr()`, `bpf_strchr()`, `bpf_strchrnul()`, `bpf_strcmp()`, `bpf_strcspn()`, `bpf_stream_vprintk()`, `bpf_strlen()`, `bpf_strnchr()`, `bpf_strnlen()`, `bpf_strnstr()`, `bpf_strrchr()`, `bpf_strspn()`, `bpf_strstr()`, and `bpf_stream_vprintk()`). 

Notably absent from the 6.17 merge window was any action on the [bcachefs pull request](<https://lwn.net/ml/all/22ib5scviwwa7bqeln22w2xm3dlywc4yuactrddhmsntixnghr@wjmmbpxjvipv>). In the contentious conversation that has followed, bcachefs developer Kent Overstreet [said](<https://lwn.net/ml/all/5ip2wzfo32zs7uznaunpqj2bjmz3log4yrrdezo5audputkbq5@uoqutt37wmvp>): ""I just got an email from Linus saying 'we're now talking about git rm -rf in 6.18'"", but no further information is available publicly. 

The 6.17 kernel now goes into the stabilization phase, with the mostly likely date for the final release being September 28.
