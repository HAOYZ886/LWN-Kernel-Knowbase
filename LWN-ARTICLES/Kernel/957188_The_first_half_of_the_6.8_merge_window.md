---
title: The first half of the 6.8 merge window
url: https://lwn.net/Articles/957188/
date: "January 12, 2024"
category: "Releases-6.8"
author: "By Jonathan Corbet January 12, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 12, 2024

The 6.8 merge window has gotten off to a relatively slow start; reasons for that include a significant scheduler performance regression that Linus Torvalds [stumbled into](<https://lwn.net/ml/linux-kernel/CAHk-=wgWcYX2oXKtgvNN2LLDXP7kXkbo-xTfumEjmPbjSer2RQ@mail.gmail.com/>) and has spent time tracking down. Even so, 4,282 non-merge changesets have found their way into the mainline repository for the 6.8 release as of this writing. These commits have brought a number of significant changes and new features. 

Some of the more interesting changes merged so far include: 

#### Core kernel

  * The [deadline servers](<https://lwn.net/Articles/934415/>) mechanism has been added as a way to prevent the starvation of normal tasks when realtime tasks are using all available CPU time. 
  * The [zswap](<https://docs.kernel.org/admin-guide/mm/zswap.html>) subsystem has gained the ability to force cold pages out to (real) swap when memory gets tight. [This commit](<https://git.kernel.org/linus/b5ba474f3f51>) includes some documentation on how to opt into or out of this feature. 

There is also a new zswap mode that disables writing back to swap entirely; see [this commit](<https://git.kernel.org/linus/501a06fe8e4c>) for details. 
  * The [DAMON](<https://docs.kernel.org/mm/damon/index.html>) memory-management facility now supports an auto-tuning mechanism; see [this changelog](<https://git.kernel.org/linus/9294a037c015>) for more information. 
  * The new `TRANSPARENT_HUGEPAGE_NEVER` configuration option causes the use of transparent huge pages to be disabled by default. 
  * Transparent huge pages can now [be allocated in multiple sizes](<https://lwn.net/Articles/937239/>) below the normal huge-page size. See [this commit](<https://git.kernel.org/linus/3485b88390b0>) for some documentation on how to control this feature. 
  * The new `UFFDIO_MOVE` operation for [`userfaultfd()`](<https://man7.org/linux/man-pages/man2/userfaultfd.2.html>) allows pages to be moved within a virtual address space; see [this commit](<https://git.kernel.org/linus/adef440691ba>) for details. 
  * The "KSM advisor" feature allows for automated tuning of the [kernel samepage merging](<https://docs.kernel.org/mm/ksm.html>) subsystem; see [this commit](<https://git.kernel.org/linus/4e5fa4f5eff6>) and [this documentation patch](<https://git.kernel.org/linus/0710f38ad26a>) for details. 
  * The BPF verifier has seen a considerable amount of work that should result in successful verification of a wider range of correct programs. 

#### Filesystems and block I/O

  * The kernel is now able to prevent direct writes to block devices that contain mounted filesystems. This feature, controlled by the `BLK_DEV_WRITE_MOUNTED` configuration option, is disabled by default but seems likely to be enabled by distributors if it is shown to not break existing workloads. Writes to devices containing mounted Btrfs filesystems remain unrestricted in any case for now, pending the merging of some support patches into that filesystem. (See [this article](<https://lwn.net/Articles/941764/>) for some background on this change). 
  * The [`listmount()` and `statmount()` system calls](<https://lwn.net/Articles/950569/>) have been merged; they allow user space to obtain detailed information about mounted filesystems. See [this changelog](<https://git.kernel.org/linus/8c9440fea774>) for more information. 
  * The XFS filesystem continues to accumulate changes adding support for the eventual online-repair feature. 
  * The SMB filesystem has gained the ability to create block and character special files. 
  * Bcachefs now has a partial (but functional) online filesystem check and repair mechanism. 

#### Hardware support

  * **Miscellaneous** : DesignWare PCIe performance-monitoring units, Intel IAA compression accelerators, Intel QAT_420xx crypto accelerators, and Lantiq PEF2256 (FALC56) pin controllers. 
  * **Networking** : Lantiq PEF2256 (FALC56) framers and Texas Instruments DP83TG720 Ethernet 1000Base-T1 PHYs. **Also** : a number of ancient wireless drivers (atmel, hostap, zd1201, orinoco, ray_cs, wl3501, rndis_wlan, and libertas 16-bit PCMCIA) have been removed. 

#### Miscellaneous

  * Rust support has been added for the creation of network PHY drivers. This work includes [a set of abstractions](<https://git.kernel.org/linus/f20fd5449ada>) making the driver API available and [a reference driver](<https://git.kernel.org/linus/cbe0e4150896>) for Asix PHYs. This is the first user-visible Rust code added to the kernel, though it duplicates the functionality of an existing driver and thus does not add new features — yet. 

#### Networking

  * There has been a fair amount of low-level work to reorganize a number of core networking data structures for better cache efficiency. This may seem like a small change but, as [the networking pull request](<https://git.kernel.org/linus/3e7aeb78ab01>) noted: ""This improves TCP performances with many concurrent connections up to 40%"". 
  * The [bpfilter](<https://lwn.net/Articles/822744/>) subsystem was meant to be a way of writing firewall rules using BPF; it was first merged for the 4.18 kernel in 2018, but never got to a point where it was usable and has seen little development in recent years. The bpfilter code [has now been removed](<https://git.kernel.org/linus/98e20e5e13d2>), though development is said to continue in [an external repository](<https://github.com/facebook/bpfilter>). The associated "usermode blob" mechanism (which was [transformed into "usermode driver"](<https://git.kernel.org/linus/e2dc9bf3f527>) in 2020) remains in the kernel, though there are no users for it. 

#### Security-related

  * There are [three new system calls](<https://lwn.net/Articles/919059/>) — `lsm_list_modules()`, `lsm_get_self_attr()`, and `lsm_set_self_attr()` \- for working with Linux security modules. See [Documentation/userspace-api/lsm.rst](<https://docs.kernel.org/next/userspace-api/lsm.html>) for details. 
  * The [BPF token mechanism](<https://lwn.net/Articles/947173/>), which allows fine-grained delegation of BPF-related permissions, was initially merged into the networking tree for inclusion in 6.8. That code ran into trouble, though, when Torvalds realized that it was still treating file descriptor zero as being special; suffice to say [he was not pleased](<https://lwn.net/ml/bpf/CAHk-=wg7JuFYwGy=GOMbRCtOL+jwSQsdUaBsRWkDVYbxipbM5A@mail.gmail.com/>). So this code was reverted for repairs; discussions are still underway and it will not be ready for this kernel release. 

#### Internal kernel changes

  * The [scope-based resource management](<https://lwn.net/Articles/934679/>) mechanism feature has gained some new guards for conditional locks (as obtained with `mutex_trylock()` and the like). See [this commit](<https://git.kernel.org/linus/e4ab322fbaaa>) for a bit more information. 
  * As expected, the venerable SLAB memory allocator has been removed, leaving SLUB as the only object-level allocator in the kernel. According to [the merge message](<https://git.kernel.org/linus/d30e51aa7b1f>): ""Removing the choice of allocators has already allowed to simplify and optimize the code wiring up the kmalloc APIs to the SLUB implementation"". 
  * The `MAX_ORDER` macro is no more; see [this article](<https://lwn.net/Articles/956321/>) for the whole story. 
  * The kernel now builds with `-Wmissing-prototypes` (which generates warnings for calls to functions that have not had a prototype declared for them) on all architectures. 

The 6.8 merge window can be expected to remain open through January 21\. Tune back in once it has closed for a summary of the remaining changes merged for the next kernel release.
