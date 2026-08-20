---
title: The first half of the 7.1 merge window
url: https://lwn.net/Articles/1067250/
date: "April 16, 2026"
category: "Releases-7.1"
author: "By Jonathan Corbet April 16, 2026"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
April 16, 2026

The 7.1 merge window opened on April 12 with the [release](<https://lwn.net/ml/all/CAHk-=wj2WqpPBwpAXo8bj_Hx-NxKMRVTVMUaQis7+Vm6XLRZiw@mail.gmail.com/>) of the 7.0 kernel. Since then, 3,855 non-merge changesets have been pulled into the mainline repository for the next release. This merge window is thus just getting started, but there has still been a fair amount of interesting work moving into the mainline. 

Some of the more interesting changes merged so far include: 

#### Architecture-specific

  * The `amd-pstate` power-management subsystem has gained support for dynamic performance preference, changing the system's power-management behavior depending on whether the system is running on AC or battery power. See [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e30ca6dd5345>) for some documentation. 
  * Support for some old and unused 486 subarchitectures (specifically M486, M486SX, and ELAN) has been removed. 
  * Intel's [Flexible Return and Event Delivery (FRED)](<https://www.intel.com/content/www/us/en/content-details/779982/flexible-return-and-event-delivery-fred-specification.html>) ""defines new control-flow transitions (generally between privilege levels) that replace existing transitions (such as event delivery through the IDT and return using IRET)."" This feature has been supported since the 6.9 release but disabled by default; as of 7.1, FRED will, instead, be enabled by default. 
  * Support for NVIDIA Tegra410 memory-latency performance-monitoring units has been added. See the documentation added in [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=429b7638b2df>) and [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2f89b7f78c50>) for details. 
  * The Arm 9.6 LSUI feature adds instructions allowing the kernel to access user-space memory without disabling the "privileged access never" mode first; 7.1 uses these instructions to accelerate futex operations. 
  * Support for Arm's [Memory Partitioning and Monitoring (MPAM)](<https://developer.arm.com/documentation/ihi0099/ba/>) feature has been improved and exposed to user space. See [Documentation/arch/arm64/mpam.rst](<https://docs.kernel.org/next/arch/arm64/mpam.html>) for more information. 
  * The BPF just-in-time compiler on PowerPC systems has improved, with support for private stacks, [fsession](<https://lwn.net/Articles/1053296/>) support, indirect jumps, and more. 
  * The s390 architecture has also gained BPF fsession support. 

#### Core kernel

  * There are three new flags to the [`clone3()`](<https://man7.org/linux/man-pages/man2/clone.2.html>) system call. `CLONE_AUTOREAP` causes a process to automatically reap itself on exit rather than becoming a zombie and waiting for the parent to do the work. `CLONE_NNP` sets the [no new privileges flag](<https://docs.kernel.org/userspace-api/no_new_privs.html>) on the newly created process. And `CLONE_PIDFD_AUTOKILL` will cause the created child to be killed immediately when the pidfd given to its parent is closed. See [this article](<https://lwn.net/Articles/1059673/>) for more information on all of these flags. 
  * There is a new file, `import_ns`, for each loaded module in its associated `/sys/module` directory; it shows which [symbol namespaces](<https://lwn.net/Articles/760045/>) the module in question has imported. 
  * The [io_uring subsystem](<https://www.man7.org/linux/man-pages/man7/io_uring.7.html>) has [gained BPF support](<https://lwn.net/Articles/1062286/>), allowing the main dispatch loop to be replaced by a BPF program. 
  * The high-resolution-timer core has been substantially rewritten for better performance; see [this merge log](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c1fe867b5bf9>) for details. Among other things, the improvements mean that the scheduler can use high-resolution timers with no performance loss relative to a scheduler using coarse timers. 
  * The project to add [proxy execution](<https://lwn.net/Articles/953438/>) to the kernel continues with the merging of part of the [donor migration](<https://lwn.net/ml/all/20251124223111.3616950-1-jstultz@google.com/>) patch set, which will eventually enable the movement of donor tasks between CPUs to facilitate the donation of CPU time to a lock holder. This work is not yet complete, but is getting closer. 
  * The way stack liveness is tracked in the BPF verifier has changed, yielding much faster verification for many programs. See [this merge log](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e2e6a6ea2418>) for more information. 

#### Filesystems and block I/O

  * The new `FSMOUNT_NAMESPACE` option to [`fsmount()`](<https://man7.org/linux/man-pages/man2/fsmount.2.html>) will cause a new namespace to be created to hold the newly mounted filesystem. The `clone3()` and [`unshare()`](<https://man7.org/linux/man-pages/man2/unshare.2.html>) system calls have also gained flags to return a new mount namespace containing only a single [nullfs](<https://lwn.net/Articles/1062163/>) mount. 
  * The kernel can now generate and verify [T10 protection information](<https://deepwiki.com/songtao-vip/linux-block/8.2-t10-protection-information>) at the filesystem level. The block layer has had this capability for a while, but moving it to the filesystems improves efficiency (especially for reads) and will facilitate the eventual addition of this support to io_uring. 
  * The [ublk](<https://docs.kernel.org/block/ublk.html>) user-space block driver has gained support for zero-copy I/O; [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=08677040a911>) contains some documentation on how to use this feature, and [this one](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=166b476b8dee>) provides a self-test for it. 
  * [SED-OPAL](<https://en.wikipedia.org/wiki/Opal_Storage_Specification>) is a specification for self-encrypting block devices. Support for SED-OPAL has been enhanced by a number of rigorously undocumented [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) operations for single-user mode support; see [this patch posting](<https://lwn.net/ml/all/20260130162527.570255-1-okozina@redhat.com/>) for an overview. 
  * The Btrfs shutdown operation, which stops all I/O operations until the subject filesystem is unmounted, was added in 6.19; in 7.1, it will lose its "experimental" status and become generally available. 
  * The exfat filesystem now supports pre-allocation with [`fallocate()`](<https://www.man7.org/linux/man-pages/man2/fallocate.2.html>). 
  * The CIFS client filesystem now supports the creation of temporary files with the `O_TMPFILE` option. 

#### Hardware support

  * **Networking** : Spacemit DWMAC Ethernet controllers, Nuvoton MA35 series Ethernet controllers, and Microchip PIC64-HPSC/HX MDIO interfaces. 

#### Networking

  * Unix-domain sockets created directly via [`socket()`](<https://man7.org/linux/man-pages/man2/socket.2.html>) (as opposed to being accessed via a filesystem) have traditionally not supported extended attributes. As of 7.1, these sockets include support for the `user.*` extended-attribute space. As described in [this merge log](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=c8db08110cbe>), the driving use case for this feature is the ability to annotate sockets with extended attributes to document the protocol that the endpoints expect to use. 
  * [UDP Lite](<https://en.wikipedia.org/wiki/UDP-Lite>) support has been removed; this move was announced in 2023 after it became clear that nobody was actually using this feature. See [this merge log](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d6e0f04bf22d>) for more information. 
  * The ability to build IPv6 support as a module has been removed; this protocol is either built directly into the kernel or disabled altogether. 

#### Security-related

  * A set of configuration options was [added](<https://git.kernel.org/linus/41e8149c8892e>) for the 6.12 release to control whether accesses to the [`/proc/_PID_ /mem` file](<https://man7.org/linux/man-pages/man5/proc_pid_mem.5.html>) could override memory permissions; by default, that overriding is allowed since that is what the kernel traditionally did. As of 7.1, though, the default will switch to `PROC_MEM_FORCE_PTRACE`, meaning that permissions can be overridden by an active [`ptrace()`](<https://man7.org/linux/man-pages/man2/ptrace.2.html>) user but not otherwise. 
  * There is a new set of security-module hooks meant to facilitate the implementation of policies for overlay filesystems. The documentation is evidently too secret for inclusion in the kernel, but [this changelog](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6af36aeb147a>) gives an overview of the hooks. 
  * There is also [a new hook](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=eb25e202b3d6>) controlling access to Unix-domain sockets in the filesystem. The Landlock security module uses this hook to [provide new policy options](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ae97330d1bd6>) for those sockets. 
  * The growing, meticulously undocumented libcrypto library, which provides faster and easier access to cryptographic algorithms than the kernel's older crypto subsystem, has gained support for a number of new algorithms; see [this merge message](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=370c38831955>) for a list. 

#### Internal kernel changes

  * The `i_ino` field of the `inode` structure is now 64 bits wide on all architectures. 
  * The way that symbols exported to modules are represented in the kernel binary [has changed](<https://lwn.net/ml/all/20260326-kflagstab-v5-0-455cd723dddf@google.com/>) to a more efficient format. 
  * The minimum version of Rust needed to build the kernel is now 1.85.0 (and bindgen 0.71.1); this change follows [the decision made at the 2025 Maintainers Summit](<https://lwn.net/Articles/1050174/>) to require the versions provided by the latest Debian stable release. See [the pull request](<https://lwn.net/ml/all/20260410200429.138694-1-ojeda@kernel.org>) for other Rust-related changes. 
  * The `kernel-doc` tool, which is used to build the kernel documentation from comments in the code, has been significantly rewritten to use a proper C tokenizer and fewer gnarly regular expressions. 
  * There have been a number of improvements around correct lock usage, including the addition of support for context analysis (formerly [capability analysis](<https://lwn.net/Articles/1012990/>)) for rwsems, mutexes, and rtmutexes. 

As of this writing, there are still nearly 9,000 non-merge changesets waiting to move from linux-next into the mainline, so expect to see a lot more interesting material in this development cycle. If the usual schedule holds, this merge window can be expected to close on April 26; tune in after that for a look at what all those changesets brought in.
