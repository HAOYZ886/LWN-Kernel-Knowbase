---
title: "XFS, LSM, and low-level management APIs"
url: https://lwn.net/Articles/767398/
date: "October 2, 2018"
category: "Filesystems-XFS; Security-Security modules; ioctl"
author: "By Jonathan Corbet October 2, 2018"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 2, 2018

The Linux Security Module (LSM) subsystem allows security modules to hook into many low-level operations within the kernel; modules can use those hooks to examine each requested operation and decide whether it should be allowed to proceed or not. In theory, just about every low-level operation is covered by an LSM hook; in practice, there are some gaps. A discussion regarding one of those gaps — low-level `ioctl()` operations on XFS filesystems — has revealed a thorny problem and a significant difference of opinion on what the correct solution is. 

In late September Tong Zhang [pointed out](<https://lwn.net/ml/linux-kernel/5EF0D46A-C098-4B51-AD13-225FFCA35D4C@vt.edu/>) that `xfs_file_ioctl()`, the 300-line function that dispatches the various `ioctl()` operations that can be performed on an XFS filesystem, was making a call to `vfs_readlink()` without first consulting the `security_inode_readlink()` LSM hook. As a result, a user with the privilege to invoke that operation (`CAP_SYS_ADMIN`) could read the value of a symbolic link within the filesystem, even if the security policy in place would otherwise forbid it. Zhang suggested that a call to the LSM hook should be added to address this problem. 

XFS developer Dave Chinner [disagreed](<https://lwn.net/ml/linux-kernel/20180926013329.GD31060@dastard/>), saying that such operations are below the level that security modules should be operating at: 

I really don't think these interfaces are something the LSMs should be trying to intercept or audit, because they are essentially internal filesystem interfaces used by trusted code and not general user application facing APIs. 

Many of the operations carried out by XFS `ioctl()` commands, such as deduplication of file contents, fast backups, and defragmentation, require bypassing all protections that apply at higher levels; an LSM just isn't relevant at this level, Chinner argued. 

Unsurprisingly, others were not entirely on board with this point of view. Stephen Smalley, the maintainer of the SELinux LSM, [said](<https://lwn.net/ml/linux-kernel/dc3bf978-1b98-bebd-6f35-9b2795693ea7@tycho.nsa.gov/>) that ""if they are interfaces exposed to userspace, then they should be mediated via LSM"". Alan Cox [added](<https://lwn.net/ml/linux-kernel/20180926192426.472360ea@alans-desktop/>) that ""in a secure environment low level complete unrestricted access to the file system is most definitely something that should be mediated"". Neither seemed to think that there was anything particularly special about `ioctl()` operations on XFS filesystems. And there could be security benefits to proper LSM coverage; Cox [described](<https://lwn.net/ml/linux-kernel/20181001160442.47c798bc@alans-desktop/>) a scenario where these operations are fully mediated: 

With a proper set of LSM checks you can lock the filesystem management and enforcement to a particular set of objects. You can build that model where for example only an administrative login from a trusted console may launch processes to do that management. 

There are, according to Chinner, a few significant problems with this particular vision of how the system should work. [One of those](<https://lwn.net/ml/linux-kernel/20180927013812.GF31060@dastard/>) is that the kernel is full of `ioctl()` operations that carry out privileged tasks. In theory, the `security_file_ioctl()` hook mediates access to those operations, but there are vast numbers of them and no security module can be expected to recognize and properly reason about even a small fraction of them. Outside of the context where a given `ioctl()` command is implemented, it is difficult to make any sense out of what that command will do or what security policies should apply to it. `ioctl()` is, by its nature, a black box that can do just about anything. 

Since there are so many of these operations, just adding LSM checks to the XFS operations, even if it could be done correctly, would not solve the problem. Chinner [pointed out](<https://lwn.net/ml/linux-kernel/20181001002521.GM31060@dastard/>) that the device-mapper `ioctl()` operations are also only protected by a `CAP_SYS_ADMIN` check, with no LSM involvement. As a result, an attacker with root privileges could simply remap blocks underneath the filesystem. Many operations at the block-device level also only check for `CAP_SYS_ADMIN`. As a result, he said: 

The storage stack is completely dependent on a simplistic layered trust model and that root (CAP_SYS_ADMIN) is god. The storage trust model falls completely apart if we don't have a trusted root user to administer all layers of the storage stack. 

This trust model has created problems at other times, he added. The `CAP_SYS_ADMIN` checks have had to be tightened up to only allow operations when the user is privileged in the initial namespace; otherwise user namespaces create problems. The issues around [unprivileged mount operations](<https://lwn.net/Articles/755593/>) also trace back to this trust model. Fixing this problem now is far from straightforward. 

There are also some practical issues; adding an LSM check now risks breaking scripts in the wild, creating regressions that would have to be reverted. Some regressions [could be severe](<https://lwn.net/ml/linux-kernel/20180927020822.GG31060@dastard/>): 

As such, there are very few trusted applications have "massive data loss" as a potential failure mode if an inappropriately configured LSM is loaded into the kernel. Breaking a HSM application's access to the filesystem unexpectedly because someone didn't set up a new security policy correctly brings a whole new level of risk to administrating sites that mix non-trivial storage solutions with LSM-based security. 

Ted Ts'o [suggested](<https://lwn.net/ml/linux-kernel/20181001152529.GA2549@thunk.org/>) that anybody who wants to control low-level filesystem operations with security modules should sit down and specify how the whole thing would work: ""a formal security model, and detail *everything* that would need to be changed in order to accomplish it"". The resulting changes, he predicted, would have to be made in ""a really huge number of places"". Chances are, this request will bring the conversation to a close in the near future. While there may be numerous developers who would like to see the system's behavior changed in this regard, most of them are likely to shy away once they realize how much work would be required to do the job in a way that would actually increase security without causing regressions. It may well be many years too late to try to add that level of security to Linux.
