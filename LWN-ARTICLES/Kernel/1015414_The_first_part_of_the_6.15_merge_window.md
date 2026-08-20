---
title: The first part of the 6.15 merge window
url: https://lwn.net/Articles/1015414/
date: "March 28, 2025"
category: "Releases-6.15"
author: "By Jonathan Corbet March 28, 2025"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
March 28, 2025

As of this writing, 6,653 non-merge changesets have been pulled into the mainline kernel repository for the 6.15 release. This merge window is thus well underway. A number of significant changes have been merged so far; read on for our summary of the first half of the 6.15 merge window. 

The most interesting changes merged to date include: 

#### Architecture-specific

  * Support for larger 32-bit x86 systems (those with more than eight CPUs or more than 4GB of RAM) has been removed. Those hardware configurations have been unavailable for a long time, and any workloads needing such resources should have long since moved to 64-bit systems. 
  * The way in which the POSIX timer subsystem assigns timer IDs [has been enhanced](<https://lwn.net/Articles/1012490/>) to allow the Checkpoint/Restore in Userspace (CRIU) subsystem to reliably and quickly restore timer IDs. 

#### Core kernel

  * The exit status of a process represented by a pidfd can be fetched even after the process has been reaped; see [this commit](<https://git.kernel.org/linus/6092c5016005>) for more information. 
  * The [special value `PIDFD_SELF`](<https://lwn.net/Articles/992991/>) can by used by a process to refer to itself in the pidfd-taking system calls. 
  * The way the kernel handles pidfd notifications in a multi-threaded process that either executes a new program or exits has changed; see [this commit](<https://git.kernel.org/linus/0fb482728ba1>) for details. 

#### Filesystems and block I/O

  * There is a new `override_creds` mount option for overlayfs filesystem that changes the credentials that are used to access the lower layers; see [this documentation commit](<https://git.kernel.org/linus/539a0879de47>) for some more information. 
  * All of the kernel's pseudo filesystems have now been converted to [the new mount API](<https://lwn.net/Articles/979166/>). Amusingly, the System V filesystem has also been converted, even though it was removed entirely later in this merge window. If that removal has to be reverted for any reason, at least the filesystem will have been updated to match current practice. 
  * There is a new API to receive information about filesystem mount and unmount events. This meticulously undocumented API is based on [the fanotify mechanism](<https://man7.org/linux/man-pages/man7/fanotify.7.html>); there are a few notes on its use in [this commit](<https://git.kernel.org/linus/0f46d81f2bce>). 
  * The [`statmount()`](<https://man7.org/linux/man-pages//man2/statmount.2.html>) system call can now receive information about the ID mappings applied to a filesystem mount. [This commit](<https://git.kernel.org/linus/37c4a9590e1e>) has some information on how the API works. 
  * It is now possible to create an ID-mapped mount of another mount that is already ID-mapped, thus changing the mappings. [This commit](<https://git.kernel.org/linus/7a54947e727b>) describes the motivation and implementation of this feature at length. 
  * There have been a number of mount-API changes to make it easier to assemble complex filesystem hierarchies without exposing partial results or parts of any filesystem that are meant to remain hidden. [This merge message](<https://git.kernel.org/linus/130e696aa68b>) contains a lot of details on what is now possible. 
  * The block layer has gained support for hardware-wrapped encryption keys. This is a mechanism that allows the kernel to program encryption keys into a block device without actually keeping the key in memory, where it might be disclosed to an attacker. [This commit](<https://git.kernel.org/linus/ebc4176551cd>) contains a documentation file describing the feature, while [this commit](<https://git.kernel.org/linus/1ebd4a3c095c>) includes documentation for the related `ioctl()` operations. 
  * The XFS filesystem now supports zoned storage devices. 
  * The EROFS filesystem now supports 48-bit block addressing to enable it to handle massive files. 
  * Bcachefs has gained a "scrub" functionality that will attempt to read all data and metadata within a filesystem, then repair, if possible, any errors found. Bcachefs is also now able to handle filesystems with a block size larger than the system page size. 

#### Hardware support

  * **Hardware monitoring** : Congatec board sensors, Texas Instruments INA233 power monitors, and Measurement Specialties HTU31 humidity and temperature sensors. 
  * **Input** : Apple touch bars. 
  * **Media** : Qualcomm iris V4L2 decoders, Synopsys DesignWare HDMI receivers, and Lontium LT6911UXE decoders. 
  * **Miscellaneous** : Sophgo SG2042 MSI interrupt controllers, Sophgo SG2042 pulse-width modulators, NXP PF9453 regulators, T-HEAD TH1520 power domains, Samsung Galaxy Book platform devices, Huawei Matebook E Go embedded controllers, Rockchip UFS host controllers, Renesas RZ/G3E system controllers, and NXP i.MX8Q reset controllers. 
  * **Networking** : [MCTP](<https://en.wikipedia.org/wiki/Management_Component_Transport_Protocol>)-over-USB interfaces, Airoha network processor units, and Realtek 8814AE and 8814AU network adapters. 
  * **Sound** : Awinic aw88166 amplifiers. 
  * **SPI** : QPIC SNAND controllers, STMicroelectronics STM32 OCTO SPI controllers, and Sophgo SG2044 SPI NOR controllers. 

#### Networking

  * Work continues toward the breaking up of the RTNL lock (the "big networking lock"), which is a contention bottleneck in much of the networking subsystem. 
  * Initial support for zero-copy data reception via io_uring has been added. 
  * There is a new TCP socket option, `TCP_RTO_MAX_MS`, that can be used to set the maximum time between retransmit attempts on an IPv4 connection. There is also a new sysctl knob to set this limit system-wide. 
  * There is a new set of BPF callbacks to obtain timestamps from various places in the networking stack; this feature is intended to help in the debugging of latency problems. This feature is severely undocumented, but [this commit](<https://git.kernel.org/linus/f4924aec58dd>) includes some self tests that show how it works. 

#### Security-related

  * There is a new security hook for the io_uring subsystem, allowing security modules more control over what is allowed; the SELinux security module has gained support for this hook. 
  * The SELinux security module can now apply policy controls to many types of data read by the kernel, including firmware images, security policies, certificates, and more. This change [drew some criticism](<https://lwn.net/ml/all/CAHk-=wjbahY4JjLCXNT4o3xVq2cejqBG69z+MNfHsN9FQBnbOw@mail.gmail.com>) from Linus Torvalds, who did not see why it was necessary. It is not clear that the subsequent conversation convinced him of its value, but the feature was merged anyway. 

#### Internal kernel changes

  * The minimum version of Python needed for code shipped with the kernel (including the documentation build system) has been raised to 3.9. 
  * The minimum GCC version (for x86 builds) is now 8.1, while the minimum Clang version is 15.0.0. 

The 6.15 merge window can be expected to remain open through April 6, after which it will be time to stabilize all of that new work. There are still nearly 6,000 commits sitting in linux-next, so the list of features for the next kernel release is far from complete. As usual, we will post another summary once the merge window closes describing those remaining changes.
