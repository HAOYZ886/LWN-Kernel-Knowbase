---
title: Overlayfs issues and experiences
url: https://lwn.net/Articles/636943/
date: "March 17, 2015"
category: "Filesystems-Union; Overlayfs"
author: "By Jake Edge March 17, 2015 LSFMM 2015"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
March 17, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

David Howells and Mike Snitzer led a discussion at the 2015 Linux Storage, Filesystem, and Memory Management (LSFMM) Summit about the [overlay filesystem](<https://lwn.net/Articles/403012/>) (overlayfs), which is the union filesystem implementation that was adopted into the kernel in 3.18. There are a number of problems that need to be addressed for this new filesystem. 

Howells was first up. He noted that overlayfs does not play nicely with security technologies that use object labels (e.g. SELinux). There are a couple of problems that he [reported](<https://lwn.net/Articles/619195/>) back in November. Overlay filesystems can have three different inodes for any given file, one in the overlayfs itself, one in the read-only lower layer, and another in the writable upper layer if the file has been written (and, thus, copied up to the upper layer). The problem for SELinux and others regards which of the three different possible versions of the inode (i.e. lower, upper, or overlay) is visible to them. That affects what security labels will be seen on the file. But those problems have largely been solved at this point. 

[ ![\[David Howells\]](https://static.lwn.net/images/2015/lsf-howells-sm.jpg) ](<https://lwn.net/Articles/636963/>)

There are two more problems, for file locking and fanotify, that still need to be addressed. The first is a Jeff Layton problem, while the other is an Eric Paris problem, Howells said with a chuckle. Layton was present, so the discussion turned to locking. What happens when an overlayfs file that has not been written to is locked (so the lock must be placed on the lower layer), then written to so that it must be copied up from the lower layer into the upper? Should the lock be copied up too? What if there are two overlays referring to the same underlying file, each of which has a copied-up version of the file, where should the lock go then? 

As it turns out, the fanotify problems are similar. If an application requests notifications on an overlayfs file that has not been written to, the notification must get placed on the lower layer inode. If the notifications are not copied up when the file gets written, then applications won't get notified even if changes are being made to the file. 

James Bottomley suggested that the semantics for file locking and fanotify need to be worked out before a mechanism to satisfy them can be proposed. Ted Ts'o was uncomfortable having different behavior based on whether the file was part of an overlayfs. Howells noted that things can get worse than he had described when you add in network filesystems (e.g. SMB or NFS) as the overlayfs layers. He noted that he had [posted](<http://permalink.gmane.org/gmane.linux.file-systems/92398>) a message in January with all of the problems he could think of, but "there are probably more". 

Layton suggested returning `ENOLCK` when trying to lock files in an overlayfs until the semantics could be worked out and implemented. Al Viro noted that with overlayfs, a file opened for reading may have a different inode number than one opened for writing. That could be a problem for a number of different applications. The classic example is a mail user agent, Viro said, but some editors also care. 

Bottomley said that there is a need to avoid surprise semantics. To do that, the developers need to know what actually matters and what users care about. POSIX semantics were broken for overlayfs, but does that really harm real users? "There is a limit to how far we need to dig to find problems that people are not complaining about", he said. 

[ ![\[Mike Snitzer\]](https://static.lwn.net/images/2015/lsf-snitzer-sm.jpg) ](<https://lwn.net/Articles/636964/>)

One of the users of overlayfs is Docker, so Snitzer wanted to look at that use case. Docker tried Btrfs, but didn't like it, he said. The project can't use block-based solutions, such as those based on device mapper and thin provisioning (thinp) that most Linux distributions use. The reason behind that is "lame" in Snitzer's view. Essentially, the project wants its Go programs to be built once (on Ubuntu), then to be able to be run on any other distribution forever, which requires statically built binaries. But there is no static library available for udev, which means that the devicemapper graph driver [cannot be used](<https://github.com/docker/docker/pull/10664>). That is a political, not a technical, issue, Snitzer said. 

The big reason that Docker has switched to overlayfs is to gain the memory efficiency that comes from pages in the page cache being shared between the containers. That doesn't happen with thinp currently, but Snitzer said that Dave Chinner has some ideas for using XFS on top of thinp to achieve it. 

Chinner spoke up to describe the problem, which is that there might be a hundred containers running on a system all based on a snapshot of a single root filesystem. That means there will be a hundred copies of glibc in the page cache because they come from different namespaces with different inodes, so there is no sharing of the data. Basically, he said, there needs to be a kind of page cache deduplication to fix the problem. 

Bottomley noted that it was a similar problem to the one that [KSM](<https://lwn.net/Articles/306704/>) tries to solve. KSM basically uses hashes of the contents of various pages of memory to share memory better between virtual machines. For containers, the main need is to deduplicate the page cache specifically. Bottomley said that the company he works for, Parallels, has a solution to the deduplication problem that does not require hashing each page, but that it is, currently at least, proprietary. Sharing of memory between containers is something that many are looking for, though, so there was some discussion of how to do it without the overhead that KSM incurs. That is where things wound down. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
