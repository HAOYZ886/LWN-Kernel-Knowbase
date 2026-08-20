---
title: "Filesystem medley: EROFS, NTFS, and XFS"
url: https://lwn.net/Articles/1055062/
date: "January 23, 2026"
category: "Filesystems-EROFS; Filesystems-NTFS; Filesystems-XFS"
author: "By Jonathan Corbet January 23, 2026"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 23, 2026

Filesystems seem to be one of those many areas where the problems are well understood, but there is always somebody working toward a better solution. As a result, filesystem development in the Linux kernel continues at a fast pace even after all these years. In recent news, the EROFS filesystem is on the path to gain a useful page-cache-sharing feature, there is a new NTFS implementation on the horizon, and XFS may be about to get an infrastructure for self healing. 

#### EROFS page-cache sharing

The [Enhanced Read-Only Filesystem (EROFS)](<https://erofs.docs.kernel.org/en/latest/>) is, as its name would suggest, a filesystem for read-only data. EROFS supports an assortment of advanced filesystem features and provides high-performance data compression. It can be found in a number of settings, perhaps most notably on Android devices. EROFS was merged for the 5.4 kernel release in 2019 and has been steadily developed since then. 

One common use case for EROFS is as a base layer for container images. As such, an EROFS filesystem can often be mounted many times on a given machine, and there can be multiple variants of a given filesystem with minor differences. Several images may contain different application mixes, but all have the same C library, for example. While EROFS will deduplicate data within a given filesystem at creation time, it cannot do that for independently created filesystems. Duplication of files across filesystems can result in multiple open files containing the same data in the system as a whole, wasting a lot of memory as that data is placed in the page cache. It would be nice to find a way to deduplicate that page-cache data and eliminate that memory waste. 

In early 2025, Hongzhen Luo posted [a patch series](<https://lwn.net/ml/all/20250301145002.2420830-1-hongzhen@linux.alibaba.com/>) implementing page-cache sharing for EROFS. Later on, Hongbo Li picked up and continued that work; the most recent posting is [version 18](<https://lwn.net/ml/all/20260123013132.662393-1-lihongbo22@huawei.com>). It works by assigning a "fingerprint" to each file within a filesystem at creation time; that fingerprint is later used to detect files containing the same data. 

Specifically, each file within an EROFS filesystem is given an extended attribute; the name of that attribute can be set by the creator, but `trusted.erofs.fingerprint` appears to be standard usage. The actual contents of the fingerprint are not defined. A logical value to use would be a cryptographic hash of the file's contents but, as Gao Xiang [said](<https://lwn.net/ml/all/8e30bc4b-c97f-4ab2-a7ce-27f399ae7462@linux.alibaba.com/>), it could simply be an integer value assigned by the image creator. It should, of course, be different for two files if they have different content. 

The fingerprint is used if the filesystem is mounted with the `inode_share` option. Another option, `domain_id`, is used to separate users; only files mounted within the same domain ID will be deduplicated. In the absence of this restriction, an attacker who is able to mount an EROFS filesystem could attach arbitrary fingerprints to files with malicious content, possibly causing other users to obtain that content rather than the files they were hoping for. 

When a fingerprinted file is opened, the EROFS code creates an internal inode that references the data within that file. The file opened by user space is redirected (within the kernel) to that deduplication inode, which is associated with the fingerprint internally. Subsequent opens of files with the same fingerprint will simply take a reference to the deduplication inode, and read operations will be redirected to that inode's backing store. As a result, all of the files will share the same folios in the page cache, eliminating the duplicate memory usage. 

In the current implementation, at least, use of fingerprints is incompatible with direct I/O. 

Depending on the workloads running within a system, page-cache sharing can reduce memory usage by as much as nearly half in the best cases. So the attractiveness of the idea is not entirely surprising. The patch series has evolved considerably over time and a lot of review concerns have been addressed, but some still remain. Christoph Hellwig, for example, [is concerned](<https://lwn.net/ml/all/20260119083251.GA5257@lst.de/>) about the security implications of this feature, and will likely need some convincing before coming around. So, while 18 revisions is quite a few, there will likely be more yet before this feature is merged. 

#### A new NTFS

[NTFS](<https://en.wikipedia.org/wiki/NTFS>) is the standard Windows filesystem format. One would think that Linux, which has long offered interoperability with as many systems as possible, would have a good NTFS implementation, but that has never really come to pass. For years, the kernel was limited to read-only support for NTFS. Linux users wanting full access to NTFS filesystems had to make do with [ntfs-3g](<https://github.com/tuxera/ntfs-3g/blob/edge/README>), running under FUSE in user space. 

That situation appeared to change with [the arrival of ntfs3](<https://lwn.net/Articles/866112/#ntfs>) in 2021. This implementation offered full NTFS access and appeared to be what the community was waiting for, despite worries expressed at the time that the level of support behind this system was less than might be desired. It was merged for the 5.15 release, and the read-only NTFS filesystem was [removed from the kernel](<https://git.kernel.org/linus/7ffa8f3d3023>) for the 6.9 release in 2024. 

The worries about the maintenance of ntfs3 have, to an extent, been realized since its merging. The ntfs3 code, having been freshly merged in 5.15, would normally be expected to see a fair rate of change in subsequent releases as bugs are fixed and more features added. In this case, though, there were exactly four ntfs3 patches in 5.16, only one in 5.17, five in 5.18, and so on. The pace picked up somewhat around 6.0, but has dropped off more recently. There have been 67 commits to `fs/ntfs3` in the last year. Many of those changes are not by its maintainer (Konstantin Komarov), but by other developers fixing up ntfs3 for changes elsewhere in the tree. 

In October 2025, Namjae Jeon surfaced with [an alternative NTFS implementation](<https://lwn.net/ml/all/20251020020749.5522-1-linkinjeon@kernel.org/>) called "ntfsplus"; the filesystem has since just been rebranded "ntfs", matching the old, read-only implementation. The work, Jeon said, was motivated by the fact that ""ntfs3 still has many problems and is poorly maintained""; ntfs is meant to be a better-maintained and more functional NTFS implementation. The [fifth revision of this patch set](<https://lwn.net/ml/all/20260111140345.3866-1-linkinjeon@kernel.org>) was posted on January 11\. 

The series begins by reverting the removal of the read-only NTFS filesystem. This code, Jeon says, ""is much cleaner, with extensive comments, offers readability that makes understanding NTFS easier"", so it makes a better base for ongoing development than the ntfs3 code base. From there, the series adds the code needed to turn the filesystem into a proper, read/write implementation of NTFS. Toward the end, the series removes the compatibility code that ntfs3 uses to emulate the old, read-only implementation. 

There are, Jeon says, a number of advantages to this version. It uses the [iomap layer](<https://docs.kernel.org/filesystems/iomap/>) to interface with the memory-management and block subsystems; ntfs3, instead, still uses the older buffer-head interface. (It should be said that there is a patch from Komorov adding iomap support to ntfs3 currently sitting in linux-next). There is an associated project adding a filesystem checker. Jeon claims that this filesystem passes many more fstests tests than ntfs3. A set of benchmarks shows better performance than ntfs3, as much as 110% better for some workloads. 

The patch set has evolved quickly over the short time it has existed on the lists, and a lot of review comments have been addressed. The new filesystem appears to be mostly feature-complete, with one notable exception: it does not yet support journaling. The ntfs3 implementation is able to play back an existing journal (though Jeon says that this feature ""in our testing did not function correctly""). The next step, according to the cover letter, will be full journaling support. 

The kernel community is normally unenthusiastic about multiple implementations of the same functionality, and would rather see incremental improvements than wholesale replacements; that places a sizable obstacle in Jeon's path. Even so, the new code appears to be winning over the filesystem developers; there are a lot of review comments, still, but they are aimed at improving the code rather than questioning its existence. To replace the NTFS implementation again would be a big step, but it appears to not be an inconceivable one. 

#### XFS self healing

The [XFS filesystem](<https://en.wikipedia.org/wiki/XFS>) tends to be associated with big systems; it is designed to scale to massive files in great number, on systems with a lot of CPUs. The organizations that run this type of system also tend to be concerned about reliability and data integrity. A lot of work has been done on XFS in those areas in recent times; one piece of that is the [XFS autonomous self-healing infrastructure](<https://lwn.net/ml/all/176897694953.202109.15171131238404759078.stgit@frogsfrogsfrogs>) from Darrick Wong. 

The kernel-based infrastructure will not, on its own, heal a problematic filesystem. It is primarily a reporting mechanism giving a user-space daemon the ability to learn about problems; the decision on how to respond to any particular problem can then be made in user space. That daemon might respond by killing and restarting a container, initiating some sort of scrub operation, or trying to correct an error in some other way. Regardless, this is a complex operation with policy decisions that are best made outside the kernel. 

The series adds a new `ioctl()` operation, `XFS_IOC_HEALTH_MONITOR`, which will return a file descriptor to a suitably privileged process. Whenever an event of note takes place, a structure will be written to that file descriptor for user space to read and react to. For the curious, [this patch](<https://lwn.net/ml/all/176852588604.2137143.13534232614166405704.stgit@frogsfrogsfrogs>) starts with a justification of the decision to output events as binary C structures rather than using some sort of higher-level protocol encoding. 

There is a wide variety of events that can be reported, starting with an "unmount" event; it indicates that the filesystem is no longer mounted, and no further events will be produced. There is a set of events for reporting metadata corruption, distinguishing between "sick" metadata (with corruption noted at run time) and "corrupt" metadata (detected by the filesystem checker). Other events report media and I/O errors. The series also adds an `ioctl()` operation to check whether a given file descriptor refers to a file on the filesystem that is being monitored; it can give the user-space daemon confidence that it is, indeed, operating on the right filesystem. 

On the user-space side, Wong has [a repository](<https://git.kernel.org/pub/scm/linux/kernel/git/djwong/xfsprogs-dev.git/log/?h=health-monitoring>) that includes an `xfs_healer` program designed to use the new interface. For the curious (and troff-capable), there is [a minimal man page](<https://git.kernel.org/pub/scm/linux/kernel/git/djwong/xfsprogs-dev.git/tree/man/man8/xfs_healer.8?h=health-monitoring>) describing this utility; we have created [a rendered version](<https://lwn.net/Articles/1055218/>) of that page that some may find a little easier on the eyes. 

The kernel series is in its sixth revision, and it would appear to be stabilizing. It may find its way into the mainline in the near future. It might take a bit longer, though, to develop the user-space code to the point that administrators will trust it to operate on a live, production filesystem.
