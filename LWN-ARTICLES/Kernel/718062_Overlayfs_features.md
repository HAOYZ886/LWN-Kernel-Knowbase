---
title: Overlayfs features
url: https://lwn.net/Articles/718062/
date: "March 29, 2017"
category: "Filesystems-Union; Overlayfs"
author: "By Jake Edge March 29, 2017 LSFMM 2017"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
March 29, 2017

* * *

[LSFMM 2017](<https://lwn.net/Articles/lsfmm2017/>)

The [overlayfs filesystem](<https://lwn.net/Articles/403012/>) is being used more and more these days, especially in conjunction with containers. Amir Goldstein and Miklos Szeredi led a discussion about recent and upcoming features for the filesystem at LSFMM 2017. 

Goldstein said that he went back to the 4.7 kernel to look at what has been added since then for overlayfs. There has been a fair amount of work in adding support for unprivileged containers. 4.8 saw the addition of SELinux support, while 4.9 added POSIX access-control lists (ACLs) and fixed file locks. 4.10 added support for cloning a file instead of copying it up on filesystems that support cloning (e.g. XFS). 

There is ongoing work on [using overlayfs to provide snapshots](<https://lwn.net/Articles/708370/>) of directory trees on XFS. It is not clear when that will be merged, but 4.11 should see the addition of a parallel copy-up operation that should speed that operation up on filesystems that do not support cloning. 

[ ![\[Amir Goldstein\]](https://static.lwn.net/images/2017/lsfmm-goldstein-sm.jpg) ](<https://lwn.net/Articles/718293/>)

Another feature that is coming, perhaps in the 4.12 time frame, is to handle the case where an application gets inconsistent data because a copy up has occurred. Szeredi explained that if an application opens a file in the lower layer that gets copied up due to a write from some other program, the application will get only old data because it will still have that lower-layer file open. There are plans to change the `read()` and `mmap()` paths to check if a file has been copied up and change the kernel's view of the file to point at the new file. 

But Al Viro was concerned that it would change a fundamental behavior that applications expect. If a world-readable file is opened, then has its permission changed to exclude the reader (which causes a copy up), the application would not expect errors at that point, but this solution would change that. Szeredi suggested that the open of the upper file could be done without permission checks, which Viro thought might work for some local filesystems, but not for upper layers on remote filesystems. 

But Bruce Fields wondered if the behavior could even be changed the way Szeredi described. There could be applications that rely on the current behavior, or else no one is really using overlayfs. Viro said that he didn't believe any applications use the behavior. But, he noted, he has broken things in the past that didn't surface and have bugs filed until years later when users actually started testing their applications with the broken kernels. 

Szeredi pointed out that these changes will make overlayfs more POSIX compliant and that there are other changes to that end that are coming. Fields is still concerned that the semantics are going to change in subtle ways over the next few years while people are actually using the filesystem. If people use it enough, there will be bugs filed from changing the behavior. But Jeff Layton said that even if it were noticed in some applications, it would be hard to argue against bringing overlayfs into POSIX compliance. 

Goldstein said that there have also been a lot of improvements in the overlayfs test suite. There is support for running those tests from xfstests, so he asked the assembled filesystem developers to run them on top of their filesystems. He also mentioned overlayfs snapshots, which kind of turns overlayfs on its head, making the upper layer into a snapshot, while the lower layer is allowed to change. Any modifications to the lower-layer objects cause a copy-up operation to preserve the contents prior to the change, while any file-creation operation causes a whiteout in the snapshot. So when the lower layer is viewed through the snapshot, it appears just as the filesystem did at snapshot time.
