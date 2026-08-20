---
title: The second half of the 5.13 merge window
url: https://lwn.net/Articles/855140/
date: "May 10, 2021"
category: "Releases-5.13"
author: "By Jonathan Corbet May 10, 2021"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 10, 2021

By the time the last pull request was acted on and [5.13-rc1](<https://lwn.net/ml/linux-kernel/CAHk-=wiWTU+=wK9pv_YG01rXSqApCS_oY+78Ztz5-ORH5a-kvg@mail.gmail.com/>) was released, a total of 14,231 non-merge commits had found their way into the mainline. That makes the 5.13 merge window larger than the entire 5.12 development cycle (13,015 commits) and just short of all of 5.11 (14,340). In other words, 5.13 looks like one of the busier development cycles we have seen for a little while. About 6,400 of these commits came in after [the first-half summary](<https://lwn.net/Articles/854536/>) was written, and they include a number of significant new features. 

Changes merged in the second half of the 5.13 merge window include: 

#### Architecture-specific

  * The arm64 architecture has [settled on](<https://git.kernel.org/linus/782276b4d0ad>) `SPARSEMEM_VMEMMAP` as the only supported [memory-management model](<https://lwn.net/Articles/789304/>). 
  * 32-Bit PowerPC systems now have support for extended BPF and the [KFENCE](<https://lwn.net/Articles/835542/>) debugging system. 
  * PowerPC systems now support [time namespaces](<https://lwn.net/Articles/766089/>). 
  * The RISC-V architecture has gained support for [kexec](<https://man7.org/linux/man-pages/man2/kexec_file_load.2.html>), crash dumps via kexec, execute-in-place, and kprobes. 
  * s390 systems can now do stack-offset randomization in system-call handlers. 

#### Core kernel

  * BPF tracing programs may now make use of task-local storage, which provides a number of performance benefits over using maps. 
  * There is a new mechanism by which BPF programs can call kernel functions directly; its initial use is in the implementation of TCP congestion-control algorithms. Functions must be explicitly whitelisted to be made available for calling from BPF. Some information can be found in [this commit](<https://git.kernel.org/linus/e6ac2450d6de>). 
  * The function tracer (ftrace) has a new `func-no-repeats` option that causes multiple, consecutive calls to a function to be coalesced into a simple count in the output. 
  * User-space page-fault handling with [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) can now manage minor faults (those where a valid page exists but a valid page-table entry does not). See [this commit](<https://git.kernel.org/linus/7677f7fd8be7>) for information on this feature and how it is meant to be used. 
  * The old (but dangerous) `/dev/kmem` special file, which provided access to the kernel's address space, [has been removed](<https://lwn.net/Articles/851531/>) at last. 

#### Filesystems and block I/O

  * The exFAT filesystem has gained support for the `FITRIM` `ioctl()` command, which is used to inform the storage device about blocks that are no longer used. 
  * The XFS filesystem now can remove space from the last allocation group in the filesystem; this is a first step toward the ability to shrink XFS filesystems in general. 
  * There is a new quota-related system call: 

```
int quotactl_path(unsigned int cmd, const char *mountpoint, qid_t id,
            		      void *addr);
```

Its behavior is similar to [`quotactl()`](<https://man7.org/linux/man-pages/man2/quotactl.2.html>), except that it expects the path to the mount point of a filesystem rather than the block special device holding that filesystem. 
  * The [fanotify](<https://man7.org/linux/man-pages/man7/fanotify.7.html>) mechanism has always restricted a number of features to privileged users, but some of those restrictions have been removed for 5.13. See [this commit](<https://git.kernel.org/linus/7cea2a3c505e>) for a cryptic description of what's allowed and what is not. 
  * The ext4 filesystem will now [overwrite directory entries](<https://lwn.net/Articles/854054/>) when files are deleted. Ext4 can now also handle filesystems that use both case folding and encryption. 

#### Hardware support

  * **Human-interface devices** : FTDI FT260 USB HID to I2C host bridges, Microsoft Surface system aggregator module HID transports, Azoteq IQS626A capacitive touch controllers, MStar msg2638 touchscreens, Ilitek I2C 213X/23XX/25XX/Lego series touch controllers, and Hycon hy46xx touchscreens. 
  * **Miscellaneous** : Silicon Labs CP2615 USB I2C adapters, HiSilicon I2C controllers, Unisoc IOMMUs, Intel Data Accelerators performance monitors, Toshiba Visconti pulse-width modulators, MediaTek Gen3 PCIe controllers, and SiFive FU740 PCIe host controllers. 
  * **Networking** : Marvell 88X2222 PHYs, Broadcom BCM6368 MDIO bus multiplexers, Actions Semi Owl Ethernet MACs, Microsoft Azure network adapters, NXP C45 TJA11XX PHYs, and Microchip KSZ8863 and KSZ8873 switches. 
  * **Pin control** : Broadcom BCM63xx GPIO controllers, Mediatek MT8195 pin controllers, Xilinx ZynqMP pin controllers, and Realtek Otto GPIO controllers. 
  * **Sound** : Realtek RT1316 codecs, Realtek RT711 and RT715 SDCA codecs, Realtek RT1019 mono class-D audio amplifiers, and MediaTek MT6359 ACCDET jack controllers. 
  * **Virtio** : new virtio drivers have been added for Bluetooth controllers and sound devices. 

#### Security-related

  * After years of work and 34 revisions, the [Landlock security module](<https://lwn.net/Articles/703876/>) has finally been merged. 

> The goal of Landlock is to enable to restrict ambient rights (e.g. global filesystem access) for a set of processes. Because Landlock is a stackable LSM, it makes possible to create safe security sandboxes as new security layers in addition to the existing system-wide access-controls. 

See [landlock.io](<https://landlock.io/>) for more information. 

#### Internal kernel changes

  * `vmalloc()` can now create huge-page mappings. 
  * There are some new functions for allocating batches of pages in a single call: `alloc_pages_bulk()` and `alloc_pages_bulk_array()`. Documentation is scarce, but some information can be found in [this commit](<https://git.kernel.org/linus/387ba26fb1cb>) and [this one](<https://git.kernel.org/linus/0f87d9d30f21>). 
  * See also [this article](<https://lwn.net/Articles/855226/>) for more information on both of the above changes. 

One feature that failed to make it this time around is the proposed [`memfd_secret()`](<https://lwn.net/Articles/835342/>) system call, which creates areas of memory that are hidden from the rest of the system (including the kernel). Andrew Morton [expressed doubts](<https://lwn.net/ml/linux-kernel/20210505120806.abfd4ee657ccabf2f221a0eb@linux-foundation.org/>) about the utility of the feature, which brought out a few potential users saying that they would like to have it. Morton [now appears to be convinced](<https://lwn.net/ml/linux-mm/20210506180126.03e1baee7ca52bedb6cc6003@linux-foundation.org/>) but, by the time that happened, it was [too late](<https://lwn.net/ml/linux-mm/CAHk-=wjHOZk_WdgsHQ2U+y5GGMWc+Psfc3jAzd4FJC9bQh7ApA@mail.gmail.com/>) for 5.13. So `memfd_secret()` looks set to make its appearance in 5.14 instead. 

If the 5.13 kernel is released after seven -rc cycles, it will come out on June 27; if a -rc8 is required, 5.13 will supply an added cause for fireworks and celebration in the US on July 4\. There will need to be a lot of testing and fixing between now and then and, if past experience holds, approximately 2,000 more commits. The merge window is done, but there is still a fair amount of work to be done to get the next kernel release out.
