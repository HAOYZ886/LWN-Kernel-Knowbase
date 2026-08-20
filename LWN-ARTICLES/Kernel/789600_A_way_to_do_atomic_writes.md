---
title: A way to do atomic writes
url: https://lwn.net/Articles/789600/
date: "May 28, 2019"
category: "Atomic IO operations; Block layer-Atomic operations"
author: "By Jake Edge May 28, 2019 LSFMM"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
May 28, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Finding a way for applications to do atomic writes to files, so that either the old or new data is present after a crash and not a combination of the two, was the topic of a session led by Christoph Hellwig at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). Application developers hate the fact that when they update files in place, a crash can leave them with old or new data—or sometimes a combination of both. He discussed some implementation ideas that he has for atomic writes for XFS and wanted to see what the other filesystem developers thought about it. 

Currently, when applications want to do an atomic write, they do one of two things. Either they use "weird user-space locking schemes", as databases typically do, or they write an entirely new file, then do an "atomic rename trick" to ensure the data is in place. Unfortunately, the applications often do not use `fsync()` correctly, so they lose their data anyway. 

[ ![\[Christoph Hellwig\]](https://static.lwn.net/images/2019/lsf-hellwig-sm.jpg) ](<https://lwn.net/Articles/789378/>)

In modern storage systems, the devices themselves sometimes do writes that are not in-place writes. Flash devices have a flash translation layer (FTL) that remaps writes to different parts of the flash for wear leveling, so those never actually do in-place updates. For NVMe devices, an update of one logical-block address (LBA) is guaranteed to be atomic but the interface is awkward so he is not sure if anyone is really using it. SCSI has a nice interface, with good error reporting, for writing atomically, but he has not seen a single device that implements it. 

There are filesystems that can write out-of-place, such as XFS, Btrfs, and others, so it would be nice to allow for atomic writes at the filesystem layer. He said that nearly five years ago there was an interesting [paper](<https://www.usenix.org/conference/fast15/technical-sessions/presentation/verma>) from HP Research that reported results of adding a special `open()` flag to indicate that atomic writes were desired. It was an academic paper that didn't deal with some of the corner cases and limitations, but had some reasonable ideas. 

In that system, users can write as much data as they want to a file, but nothing will be visible until they do an explicit commit operation. Once that commit is done, all of the changes become active. One simple way to implement this would be to handle the commit operation as part of `fsync()`, which means that no new system call is required. 

A while back, he started implementing atomic writes using this scheme in XFS. He [posted](<https://lwn.net/Articles/715918/>) some patches, but there were multiple problems there; he has since reworked that patch set. Now the authors of the paper are "pestering him" to get the code out so that they can write another paper about it with him. Others have also asked for the feature, he said. 

Chris Mason asked what the granularity is; is it just a single `write()` call or more than that? Hellwig said that it is all of the writes that happen until the commit operation is performed. Filesystems can establish an upper bound on the amount of data that can be handled; for XFS it is based on the number of discontiguous regions (i.e. extents) that the writes touch. 

This feature would work for `mmap()` regions as well, not just traditional `write()` calls. For example, Hellwig noted that it is difficult to do an atomic update of a, say, B-Tree that updates multiple nodes. With this feature, the application can just make the changes in the file-backed memory, then do the commit; if there is a crash, they will end up with one version or the other. 

Ted Ts'o said that he found it amusing because someone he is advising on the Android team wants a similar feature, but wants it on a per-filesystem basis. The idea is that, when updating Android from one version to another, the ext4 or F2FS filesystem would be mounted with a magic option that would stop any journal commits from happening. An `ioctl()` command would then be sent once the update has finished and the journal commits would be processed. It is "kind of ugly", he said, but it gives him perhaps 90% of what would be needed to implement the atomic write feature. Toward the end of the session, Ts'o said that he believes ext4 will get the atomic write feature as well, though it will be more limited in terms of how much of the file can be updated prior to a commit. 

Hellwig expressed some skepticism, noting that he had tried to do something similar by handling the updates in memory, but that became restrictive in terms of the amount of update data that could be handled. Ts'o said that for Android, the data blocks are being written to the disk, it is just the metadata updates that are being held for the few minutes required to do the update. It is a "very restrictive use case", Ts'o said, but the new mechanism replaces a device-mapper hack that was far too slow. 

Chris Mason said that, depending on the interface, he would be happy to see Btrfs support it. Hellwig said that it should be fairly straightforward to do in Btrfs. One of the big blockers for him at this point is the interaction with `O_DIRECT`. If an application writes data atomically, then reads it back, it better get what it just wrote; no "sane application" would do that, he said, but NFS does. The Linux I/O path is not really set up to handle that, so he has some work to do there. 

There was some discussion of using `fsync()` instead of a dedicated system call or other interface. Hellwig sees no reason not to use `fsync()` since it has much the same meaning; there is no reason to do one operation without the other, he said. Amir Goldstein asked about the possibility of another process using an `fsync()` on the file as a kind of attack. 

Hellwig said that originally he was using an `open()` flag, but got reminded again that unused flags are not checked by `open()` so using a flag for data integrity is not really a good idea. Under that model, though, an `fsync()` would only map to the commit operation for file descriptors that had been opened with the flag. He has switched to an inode flag, which makes more sense in some ways, but it does leave open the problem of unwanted `fsync()` calls.
