---
title: Network filesystem topics
url: https://lwn.net/Articles/754506/
date: "May 21, 2018"
category: "Filesystems-Network"
author: "By Jake Edge May 21, 2018 LSFMM"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 21, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

At the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM), Steve French led a discussion of various problem areas for network filesystems. Unlike previous sessions (in [2016](<https://lwn.net/Articles/685431/>) and [2017](<https://lwn.net/Articles/718804/>)), there was some good news to report because the long-awaited [`statx()` system call](<http://man7.org/linux/man-pages/man2/statx.2.html>) was released in Linux 4.11. But there is still plenty of work to be done to better support network filesystems in Linux. 

French said that `statx()` was a great addition that would help multiple filesystems that do not use local block devices for their storage; that includes Samba using SMB 3.1.1 and NFS 4.2. The "birth time" (or creation time) attribute is "super important" for Samba, he said. The next step is to get more of the [Windows attribute bits](<https://en.wikipedia.org/wiki/File_attribute#DOS_and_Windows>) supported in `statx()` and also in the `FS_IOC_[GS]ETFLAGS` `ioctl()` commands. 

[ ![\[Steve French\]](https://static.lwn.net/images/2018/lsf-french-sm.jpg) ](<https://lwn.net/Articles/754534/>)

There are numerous features that Windows provides, but Linux does not, which makes life more difficult for network filesystems. There is no way to do safe caching of file and directory data because leases and delegations are not supported on Linux servers. Also, there still is no support for [rich access-control lists](<https://lwn.net/Articles/661357/>) (RichACLs) despite lots of work and testing that went on over the years. There has not been much patch activity lately, he said, but Andreas Gruenbacher has posted 28 versions of the [patch set](<https://lwn.net/Articles/714386/>) over time. The problems that have cropped up are generally due to trying to map user IDs and the like between three separate domains (perhaps server, client, and on-disk, though French did not say). 

Broader support for the variants of the fast copy operation is badly needed, he said. The `cp --reflink` command uses the [`FICLONERANGE` `ioctl()` command](<http://man7.org/linux/man-pages/man2/ioctl_ficlonerange.2.html>), but not [`copy_file_range()`](<http://man7.org/linux/man-pages/man2/copy_file_range.2.html>); in fact, no utilities use `copy_file_range()`, though it should be the default. It will fall back to other forms of copying, if needed, but can make the copy operation complete thousands of times faster in many cases. French said he got an email from a user asking about a copy operation in the cloud that was taking an hour or so. He suggested using a different command, which was faster, but the customer asked why `cp` (and other tools such as `rsync`) did not simply use the faster operation. 

Case-insensitive lookups are another problem area; Samba emulates it, but it is expensive to do so. Ric Wheeler noted that XFS supports doing case-insensitive lookups while preserving the case of the filenames on disk; he suggested perhaps doing the same in user space for Samba. French said that might make sense as this problem has been around for a long time. 

In general, macOS and Windows are both SMB friendly, but Linux is not, he said. Though he did describe a demo at a recent storage conference, where different clients on a "bad hotel network" were all able to edit the same file using SMB. It was rather eye-opening, especially when compared to ten years ago, to see Linux, macOS, Windows, Android, and iOS all interoperating that way. 

Many of the standard utilities are not transferring data in large enough chunks. For example, `rsync` defaults to 4KB and the largest it will use is 128KB, but NFS is able to handle much larger transfers and SMB is larger still. For the network filesystems, transferring 8MB chunks would make much more sense. 

He mentioned a double handful of other features that would make things easier for Samba, NFS, and others, but it was not clear who was working on those features or planning to do so—something that is also true for some of the features mentioned earlier. For example, Dave Chinner said that someone needs to update `cp` to bring it into the `copy_file_range()` world. French said that he had sent some patches to the `rsync` maintainers (who may well be easier to find than `cp` maintainers), but that there was no response. The upshot was that network filesystems, especially those that are meant to interoperate with Windows, are not getting the attention that they need from the Linux world.
