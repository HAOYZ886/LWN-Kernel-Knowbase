---
title: The beginning of the 6.19 merge window
url: https://lwn.net/Articles/1048869/
date: "December 4, 2025"
category: "Releases-6.19"
author: "By Jonathan Corbet December 4, 2025"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 4, 2025

As of this writing, 4,124 non-merge commits have been pulled into the mainline repository for the 6.19 kernel development cycle. That is a relatively small fraction of what can be expected this time around, but it contains quite a bit of significant work, with changes to many core kernel subsystems. Read on for a summary of the first part of the 6.19 merge window. 

The most significant changes merged so far include: 

#### Architecture-specific

  * Support has been added for AMD's "smart data cache injection" feature, which allows I/O devices to place data directly in the L3 cache without the need to go through RAM first. Documentation for this feature's control knobs has been dispersed throughout [Documentation/filesystems/resctrl.rst](<https://docs.kernel.org/next/filesystems/resctrl.html>); interested readers can find it by searching for "`io_alloc`". 
  * Nearly three years after LWN first [looked at the patches](<https://lwn.net/Articles/919683/>), basic support for Intel's linear address-space separation (LASS) feature has been merged. LASS implements the separation between the kernel and user-space address ranges in the hardware itself, improving security and closing off the sort of side channels that can be used by speculative attacks. In 6.19, LASS support will be disabled on most systems while some remaining problems are worked out. 
  * The s390 architecture has a new interface for the configuration and management of hotplug memory; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ec9b3b85ea28>) for an overview. 
  * Support for 31-bit binaries in compatibility mode on s390 systems has been removed; from [the merge message](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=0d79affa31cb>): ""To the best of our knowledge there aren't any 31 bit binaries out in the world anymore that would matter for newer kernels or newer distributions"". 
  * S390 systems have long lacked stack-protector support; that will change in 6.19 thanks to better support in the upcoming GCC 16 compiler release. See [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=f5730d44e05e>) for some background. 
  * Support for the Arm [Memory System Resource Partitioning and Monitoring (MPAM)](<https://neoverse-reference-design.docs.arm.com/en/latest/features/mpam.html>) capability has been added for 64-bit systems. From the [KConfig commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d8bf01d80919>): 

> Memory System Resource Partitioning and Monitoring (MPAM) is an optional extension to the Arm architecture that allows each transaction issued to the memory system to be labeled with a Partition identifier (PARTID) and Performance Monitoring Group identifier (PMG). 
> 
> Memory system components, such as the caches, can be configured with policies to control how much of various physical resources (such as memory bandwidth or cache memory) the transactions labelled with each PARTID can consume. Depending on the capabilities of the hardware, the PARTID and PMG can also be used as filtering criteria to measure the memory system resource consumption of different parts of a workload. 

This feature is strenuously undocumented within the kernel, but some information on its use can be found on [this page](<https://developer.arm.com/documentation/108032/0100/A-closer-look-at-MPAM-software/Linux-MPAM-overview>). 

#### Core kernel

  * The new [`listns()` system call](<https://lwn.net/Articles/1043824/>) allows user space to iterate through the namespaces on the system. The handling of reference counts for namespaces has also changed, making it impossible for user space to resurrect a namespace that is at the end of its life. 
  * If a process dumps core as the result of a signal, another process holding a pidfd for the signaled process can now learn which signal brought about the end. 
  * Support for deferred unwinding of user-space call stacks has been merged; this is part of the larger project of enabling support for unwinding using SFrame. See [this article](<https://lwn.net/Articles/1029189/>) for details. 
  * The restartable sequences implementation [has been massively reworked](<https://lwn.net/Articles/1033955/>), leading to better performance throughout. 
  * BPF programs [can now contain indirect jumps](<https://lwn.net/Articles/1017439/>), which are accessed by way of a dedicated "instruction set" map type. Some information on the new maps can be found in [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b4ce5923e780>), while information on indirect jumps is in [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=493d9e0d6083>). This feature is x86-only for now. 

#### Filesystems and block I/O

  * The [filesystem in userspace](<https://www.kernel.org/doc/html/next/filesystems/fuse/>) (FUSE) subsystem has improved support for buffered reads when large folios are in use. The [iomap](<https://www.kernel.org/doc/html/next/filesystems/iomap/>) layer has gained the ability to track folios that are partially up to date so that reads on such folios only need to bring in the data that is missing. 
  * The virtual filesystem layer has gained the ability to create recallable directory delegations, intended to support [NFS directory delegations](<https://datatracker.ietf.org/doc/html/rfc8881#name-delegation-and-callbacks>). This feature allows an NFS server to hand responsibility for the management of a directory to a client. That delegation can be recalled should a second client also try to make changes in the delegated directory. 
  * There is a new "file dynptr" abstraction that allows BPF programs to read data from structured files. Documentation is scarce, but [this page](<https://docs.ebpf.io/linux/concepts/dynptrs/>) covers the dynptr concept in general, and [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=784cdf931543>) contains tests for the new feature. 
  * The Btrfs filesystem now implements a "shutdown" state that attempts to complete outstanding operations but rejects any new operations. There is [an `ioctl()` operation](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=6b1ac78dd0f2>) to enter this state, but it is marked as experimental for this release. 
  * The ext4 implementation can now manage filesystems with [a block size greater than the system page size](<https://lwn.net/Articles/945646/>). 

#### Hardware support

  * **Clock** : Realtek system timers. 
  * **Miscellaneous** : Intel integrated memory/IO hub memory controllers and NXP i.MX91 thermal monitoring units. 
  * **Networking** : Eswin EIC7700 Ethernet controllers, Motorcomm YT9215 Ethernet switches, Mucse 1GbE PCI Express adapters, MaxLinear GSW1xx Ethernet switches, and Realtek 8852CU USB wireless network adapters. 

#### Miscellaneous

  * The documentation build system [has been reworked](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3df5affb4be2>), replacing a bunch of makefile logic with a separate build-wrapper script written in Python. 

#### Networking

  * A [locking change](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=2df75cc5bdc4>) deep within the TCP transmit code has yielded ""a 300 % (4x) improvement on heavy TX workloads, sending twice the number of packets per second, for half the cpu cycles"". 
  * It is now possible to mark network sockets as exempt from the system's global memory limits; the idea is that limits will be imposed within a container instead. [This commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7c268eaeec63>) describes the reasoning behind this feature, and [this one](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b46ab63181ff>) introduces a sysctl knob to control the feature. There is also [a mechanism](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=38163af06881>) to allow BPF programs to control the accounting flag. 
  * The system now supports [RFC 5837](<https://datatracker.ietf.org/doc/html/rfc5837>), providing more information via ICMP that will improve route tracing. See [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=76c231b3c2e0>) for more information. 
  * There is improved support for continuous busy polling within network drivers; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=ff371a7e73c8>) for details. 
  * The [CAN XL protocol](<https://www.can-cia.org/can-knowledge/can-xl>) is now supported. 
  * The [`getsockname()`](<https://man7.org/linux/man-pages/man2/getsockname.2.html>) and [`getpeername()`](<https://man7.org/linux/man-pages/man2/getpeername.2.html>) system calls are now supported by io_uring. 

#### Security-related

  * The kernel's internal cryptographic library has gained support for the SHA-3 and BLAKE2b hash algorithms. [This commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=059344724804>) includes documentation for the SHA-3 implementation. 
  * Security modules will now be notified when a memfd is created, allowing policies specific to those files to be implemented; SELinux has gained support for this new hook. See [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=094e94d13b60>) for more information. 
  * The [Integrity Policy Enforcement security module](<https://docs.kernel.org/next/admin-guide/LSM/ipe.html>) has gained support for the [`AT_EXECVE_CHECK` flag](<https://docs.kernel.org/userspace-api/check_exec.html>) for [`execveat()`](<https://man7.org/linux/man-pages/man2/execveat.2.html>). This flag will cause integrity checks to be performed on a script file before it is passed to an interpreter for execution. 

#### Internal kernel changes

  * The inode `i_state` field has been hidden and must now be manipulated using accessor functions; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=11f2af2a80b5>) for an overview of the new way of doing things. 
  * There are a couple of new filesystem-layer primitive operations, `FD_ADD()` and `FD_PREPARE()`, that handle a lot of the boilerplate of creating and installing files. [This commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=011703a9acd7>) provides an overview. 
  * The new `scoped_seqlock_read()` macro simplifies common uses of seqlocks; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=cc39f3872c08>) for some more information. 
  * The `objtool` utility has been reworked to facilitate the creation of live patches; see [this commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=00a155c691be>) for details. 
  * The [scoped user-space access primitives](<https://lwn.net/Articles/1042711/>) have been merged. They provide a way to access user-space data in hot code paths that is safe from speculative attacks. 

Before getting into the mainline, this code first had to work around a deficiency in both the GCC and Clang compilers: a jump instruction in assembly code that jumps out of the scope will bypass the cleanup code (which is the whole reason for using scopes in the first place). [This commit](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=3eb6660f26d1>) shows the workaround: a two-step jump, with the assembly goto remaining within the scope. 
  * The `at_least` marker, which can indicate a minimum allowable size for an array passed into a function, has been added. See [this article](<https://lwn.net/Articles/1046840/>) for more information. 
  * The directory `tools/lib/python` has been established as a central location for Python modules used by code shipped with the kernel. For now, its use is mostly limited to the documentation subsystem, but there are other Python utilities in the kernel that may eventually benefit from a move there as well. 
  * The new mempool function [`mempool_alloc_bulk()`](<https://docs.kernel.org/next/core-api/mm-api.html#c.mempool_alloc_bulk>) can be used to safely allocate multiple objects in a single call. 
  * The new `%pt5p` specifier for `printk()` can be used to print a `timespec` structure in a human-readable form. 
  * The large "syn" Rust crate is now packaged with the kernel source; it is a library for the parsing of Rust code into a syntax tree. See [this merge message](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=54e3eae85562>) for more information. 
  * Support for [`struct sockaddr_unsized`](<https://lwn.net/Articles/1045453/>), which is intended to improve memory safety for the `struct sockaddr` flexible array, has been merged. 

There are, as of this writing, still over 8,000 commits waiting in linux-next; most of those can be expected to spill into the mainline over the remainder of the merge window. Normally, the merge window would be expected to close on December 14, but Linus Torvalds suggested in [the 6.18 release announcement](<https://lwn.net/ml/all/CAHk-=whnC+hRftevTLeVs3tyyqwn+7un=jUES2-WX+pZhDdKNw@mail.gmail.com/>) that the closing might be delayed somewhat. Your editor, suffering through the third conference-overlapping merge window this year, suspects that the usual schedule will hold, though. Either way, we will provide the usual summary once the merge window closes.
