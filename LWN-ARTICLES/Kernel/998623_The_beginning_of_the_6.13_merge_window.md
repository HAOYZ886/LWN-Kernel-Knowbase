---
title: The beginning of the 6.13 merge window
url: https://lwn.net/Articles/998623/
date: "November 21, 2024"
category: "Releases-6.13"
author: "By Jonathan Corbet November 21, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
November 21, 2024

As of this writing, just over 1,800 non-merge changesets have been pulled into the mainline kernel for the 6.13 release. That number may seem small, given that a typical merge window brings in at least 12,000 commits, but the early pulls this time around have focused on significant core changes, and there are quite a few of them. The time has come to summarize the changes pulled so far, including lazy preemption, multi-grained timestamps, new extended-attribute system calls, and more. 

Some of the most significant changes pulled in the first part of the 6.13 merge window include: 

#### Architecture-specific

  * The arm64 architecture can now run Linux in virtual machines under the [Arm Confidential Compute Architecture](<https://www.arm.com/architecture/security-features/arm-confidential-compute-architecture>). 
  * Arm64 also now supports user-space shadow stacks with the [Guarded Control Stack](<https://www.kernel.org/doc/html/next/arch/arm64/gcs.html>) feature. 
  * The s390 architecture has gained support for the resizing of virtual machines with [virtio-mem](<https://virtio-mem.gitlab.io/>). There is an overview in [this commit message](<https://git.kernel.org/linus/38968bcdcc1d>). 
  * [Split-lock detection](<https://lwn.net/Articles/911219/>) is now supported on AMD CPUs. 
  * There is now [support](<https://git.kernel.org/linus/322a90638768>) for MIPS multi-cluster interrupt controllers. 

#### Core kernel

  * The [`PIDFD_GET_INFO` `ioctl()` operation](<https://lwn.net/Articles/992991/>), which will fetch information about a process represented by a pidfd, has been merged. 
  * The io_uring subsystem has a new command, `IORING_REGISTER_RESIZE_RINGS`, that allows on-the-fly resizing of the submission and completion rings. This allows applications to start with a pair of relatively small rings, and grow them later should the need arise. 
  * The [lazy preemption](<https://lwn.net/Articles/994322/>) patches have been merged. This work greatly simplifies the kernel's preemption logic (and configuration options) while maintaining performance for throughput-oriented configurations. It is a significant change that should, eventually, reduce the amount of scheduling-related logic scattered through the non-scheduler parts of the kernel. 
  * Some preliminary work needed to implement [proxy execution](<https://lwn.net/Articles/953438/>), an improved approach to the avoidance of priority inversion, has been merged. The proxy execution feature itself, though, has not yet landed. 

#### Filesystems and block I/O

  * There have been a few tries to implement fine-grained timestamps for file metadata; the last one [ran aground](<https://lwn.net/Articles/946394/>) at the end of 2023. Another attempt is being made for 6.13; [this merge message](<https://git.kernel.org/linus/6ac81fd55e8a>) contains the details of how it works. 

In short: as before, filesystems only need to track fine-grained change times for a given file if the time is being actively queried; most of the time, low-resolution timestamps are good enough. That is important, since lower-resolution timestamps do not need to be written back to persistent storage as frequently. The previous implementation ran into problems, though, where a low-resolution timestamp could appear to be earlier than a high-resolution timestamp, even though the actual changes happened in the opposite order. 

In the new implementation, the kernel remembers the last fine-grained timestamp that was given out and ensures that any coarse-grained timestamps assigned for file modifications are later than the that last fine-grained value. This technique avoids the above-mentioned problem, ensuring that timestamps always correctly reflect the order in which files were modified. 

See [this documentation commit](<https://git.kernel.org/linus/e3fad0376d80>) for more information. 
  * There is a new sysctl knob, `fs.dentry-negative`, that controls whether the virtual filesystem (VFS) layer deletes a file's kernel-internal directory entry ("dentry") when the file itself is deleted. It seems that some benchmarks do better when dentries are removed, while others benefit from having a negative dentry left behind, so the kernel developers have put the decision into the system administrator's hands. The default value (zero) means that dentries are not automatically deleted, matching the behavior of previous kernels. 
  * The [`statmount()`](<https://lwn.net/Articles/950569/>) system call has gained options to return the filesystem subtype, superblock source, and security mount options. There is also a new flag, `STATMOUNT_OPT_ARRAY`, that returns filesystem options as a series of NUL-separated strings and without the usual "`\000`" escaping. 
  * There have been some deep reference-counting changes within the VFS layer that yield a 3-5% performance improvement on highly threaded workloads; see [this merge message](<https://git.kernel.org/linus/62eec753cae2>) for some details. 
  * It is now possible to assemble an [overlayfs](<https://docs.kernel.org/filesystems/overlayfs.html>) stack using file descriptors rather than path names; see [this merge message](<https://git.kernel.org/linus/a29835c9d0ba>) for details. 
  * The tmpfs filesystem can now be mounted in a case-folding mode where file names are no longer case-sensitive. See [this documentation commit](<https://git.kernel.org/linus/a713f830c903>) for the relevant mount options. 
  * Limited support for [atomic write operations](<https://lwn.net/Articles/974578/>) has been added to the ext4 and XFS filesystems. 
  * There is a new set of system calls for the management of extended attributes: `setxattrat()`, `getxattrat()`, `listxattrat()`, and `removexattrat()`. They are variants of [`setxattr()`](<https://man7.org/linux/man-pages/man2/setxattr.2.html>), [`getxattr()`](<https://man7.org/linux/man-pages/man2/getxattr.2.html>), [`listxattr()`](<https://man7.org/linux/man-pages/man2/listxattr.2.html>), and [`removexattr()`](<https://man7.org/linux/man-pages/man2/removexattr.2.html>) that include a directory file descriptor as the starting point for the path-name search. 
  * The new `BTRFS_IOC_SUBVOL_SYNC_WAIT` `ioctl()` command for the Btrfs filesystem will wait for the cleaning of one or more subvolumes. It is an unprivileged operation, and is intended to allow the "`btrfs subvolume sync`" command to work without privilege. 
  * Btrfs now supports performing encoded reads (reading of compressed extents directly, without decompression) via io_uring. 

#### Hardware support

  * **Hardware monitoring** : Renesas ISL28022 power monitors and Nuvoton NCT7363Y fan controllers. 
  * **Miscellaneous** : Marvell PEM performance-monitoring units, Airoha true HW random number generators, Broadcom BCM74110 random number generators, Renesas RZ/V2H(P) interrupt control units, and THEAD C9XX ACLINT S-mode IPI interrupt controllers. 

#### Miscellaneous

  * There is a new user-space API allowing administrators to set thermal thresholds on specific devices; notifications will be generated when a threshold is crossed. [This commit](<https://git.kernel.org/linus/445936f9e258>) gives an overview of the functionality, but the actual (netlink-based) API is uncompromisingly undocumented. 

#### Security-related

  * The SELinux security module can now manage policies for individual netlink operations; see [this commit message](<https://git.kernel.org/linus/d1d991efaf34>) for a terse overview. 
  * The `/sys/fs/selinux/user` configuration knob [has been deprecated](<https://git.kernel.org/linus/d7b6918e22c7>) and will be removed in a future release. 

#### Internal kernel changes

  * There are now Rust abstractions for a number of VFS data structures and interfaces — enough to support the [Rust implementation of binder](<https://lwn.net/Articles/953116/>). 
  * The bulk of the [file-descriptor memory-safety work](<https://lwn.net/Articles/985853/>) has been merged. 
  * The kernel's cryptographic subsystem has gained a new internal API for signature generation. There is [some kerneldoc documentation](<https://docs.kernel.org/next/crypto/api-sig.html#asymmetric-signature-api>) available. 
  * There is a new variant of the sleepable RCU ("SRCU") API that makes the read side cheaper at the cost of more expensive write operations. Documentation for the new functions — `srcu_read_lock_lite()` and `srcu_read_unlock_lite()` — can be found by reading through [this commit](<https://git.kernel.org/linus/6364dd8191d2>). 
  * The [debugobjects subsystem](<https://docs.kernel.org/core-api/debug-objects.html>) has been massively reworked for better performance and robustness; see [this merge message](<https://git.kernel.org/linus/fb1dd1403c7b>) for details. 
  * The venerable `dontdiff` file [has been removed](<https://git.kernel.org/linus/b27f9e8079bf>) from the documentation tree. 

The 6.13 merge window can be expected to remain open through December 1\. That closing date is immediately after a significant holiday weekend in the US, but past experience suggests that the 6.13-rc1 release will come out on schedule regardless. LWN will have an update of the remaining changes from this merge window once it closes.
