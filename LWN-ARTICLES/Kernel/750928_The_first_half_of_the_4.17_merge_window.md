---
title: The first half of the 4.17 merge window
url: https://lwn.net/Articles/750928/
date: "April 5, 2018"
category: "Releases-4.17"
author: "By Jonathan Corbet April 5, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
April 5, 2018

As of this writing, 5,392 non-merge changesets have been pulled into the mainline repository for the 4.17 release. The 4.17 merge window is thus off to a good start, but it is far from complete. The changes pulled thus far cover a wide part of the core kernel as well as the networking, driver, and filesystem subsystems. 

Some of the more significant changes merged so far are: 

#### Core kernel

  * The ever-expanding [`perf_event_open()`](<http://man7.org/linux/man-pages/man2/perf_event_open.2.html>) system call has gained the ability to place a kprobe or uprobe and create an event associated with it; the probe continues to exist for the life of the resulting file descriptor. Probes created this way will not be visible in the tracefs virtual filesystem. This interface was created as a way of ensuring that probes will be cleaned up when the process that created them exits. 
  * The scheduler's load estimation code [has been improved](<https://lwn.net/Articles/741171/>), especially for mobile and embedded workloads. 
  * The new `BPF_RAW_TRACEPOINT` command for the `bpf()` system call attaches a BPF program to a tracepoint but performs no processing of the tracepoint arguments before calling that program. That enables tracing with minimal overhead, but requires more awareness when writing BPF programs. See [this commit](<https://git.kernel.org/linus/c4f6699dfcb8558d138fe838f741b2c10f416cf9>) for a terse overview of the feature. 

#### Architecture-specific

  * Support for the Andes Technologies NDS32 architecture has been added. 
  * As described in [this article](<https://lwn.net/Articles/748074/>), support for the blackfin, cris, frv, m32r, metag, mn10300, score, and tile architectures has been removed; it seems that none of them had any remaining users. The [merge commit](<https://git.kernel.org/linus/f5a8eb632b562bd9c16c389f5db3a5260fba4157>) for this removal shrinks the kernel by almost 470,000 lines of code. 
  * The SPARC "application data integrity" feature is now supported; it allows the application of tags to virtual-memory addresses. The tag is a four-bit value that is placed in the upper bits of the address; any reference lacking the proper tag will generate a trap. See [this commit](<https://git.kernel.org/linus/88fe35293446d19c4870e581b8b78d4714fc63d2>) for details. 

#### Filesystems

  * The XFS filesystem now supports the `lazytime` mount option. 
  * The Btrfs filesystem has long supported a set of `ioctl()` operations to control transactions. It has been concluded that these operations are unused, so they have been removed for 4.17. 
  * The CIFS filesystem now supports [SMB 3.1.1 pre-authentication integrity](<https://blogs.msdn.microsoft.com/openspecification/2015/08/11/smb-3-1-1-pre-authentication-integrity-in-windows-10/>). SMB 3.1.1 is also no longer marked "experimental". 
  * The ext4 filesystem has been made more robust against maliciously crafted filesystem images. Maintainer Ted Ts'o [warns](<https://git.kernel.org/linus/3e968c9f1401088abc9a19ae6ff571644d37a355>) ""I still don't recommend that container folks hold any delusions that mounting arbitrary images that can be crafted by malicious attackers should be considered sane thing to do, though!"" 

#### Networking

  * The [reliable datagram socket](<https://linux.die.net/man/7/rds>) protocol now supports [zero-copy](<https://lwn.net/Articles/726917/>) operation. 
  * It is now possible to apply BPF scripts to filter traffic sent by the `sendmsg()` and `sendfile()` system calls. See [this commit](<https://git.kernel.org/linus/d48ce3e5ba741428ed38a665a3c6b41e6cd999be>) for details. 
  * A receive-side implementation of the TLS protocol has been added. This adds to the existing [transmit implementation](<https://lwn.net/Articles/666509/>) to give full in-kernel TLS protocol support. 
  * A set of control-group-specific BPF hooks has been added to the `bind()` and `connect()` system calls; attached programs can modify how those calls work. Some information can be found in [this commit](<https://git.kernel.org/linus/7828f20e3779e4e85e55371e0e43f5006a15fb41>). 

#### Hardware support

  * **Graphics** : ARM Versatile panels, Allwinner DesignWare HDMI phys, AMD Vega12 GPUs, and Raydium RM68200 720x1280 DSI panels. 
  * **Media** : NXP TDA1997x HDMI receivers, Omnivision ov2685 and ov5695 sensors, Renesas capture engine units, Sony CXD2880 DVB-T2/T tuner/demodulators, and SoundGraph iMON receivers. 
  * **Miscellaneous** : Marvell 88PG86X voltage regulators, Aspeed KCS IPMI interfaces, and Amiga Gayle PATA controllers. 
  * **Networking** : NXP MCR20AVHM transceivers, Microchip LAN743x gigabit Ethernet interfaces, and Intel Ethernet connection E800 series interfaces. 
  * **Pin control** : Qualcomm SDM845 pin controllers, NXP IMX6SLL pin controllers, MediaTek MT2712 pin controllers, and Allwinner H6 pin controllers. 

**Miscellaneous**

  * The `perf script` command now supports Python 3 scripts. 
  * The [Linux kernel memory model](<https://lwn.net/Articles/718628/>) is now a part of the kernel proper. It includes a formal description of how memory coherency works in the kernel, along with an extensive set of tests to prove adherence to the model. See [this commit](<https://git.kernel.org/linus/1c27b644c0fdbc61e113b8faee14baeb8df32486>) for an overview of what has been merged. 

#### Internal kernel changes

  * Building the kernel for x86 now requires a compiler with `asm goto` support. In practice that means that GCC 4.5 or later is now needed. It also rules out the use of Clang on x86 until that compiler gains `asm goto` support. 
  * `wait_on_atomic_one()` has been replaced by `wait_event_var()`, a more general interface; see [this article](<https://lwn.net/Articles/750774/>) for details. 
  * A massive kernel-wide cleanup has removed almost all direct invocations of system-call implementations from within the kernel. This is useful for a number of reasons: it adds flexibility to the system-call interface and makes it easier to [remove `set_fs()` calls](<https://lwn.net/Articles/722267/>), among other things. A [significant change to the x86 system-call mechanism](<https://lwn.net/Articles/750536/>) is in the works for the near future. 

If this turns out to be a normal two-week merge window, it can be expected to remain open until April 15, with the final 4.17 release happening in early June. The remainder of the changes merged for this release will be summarized once the merge window closes.
