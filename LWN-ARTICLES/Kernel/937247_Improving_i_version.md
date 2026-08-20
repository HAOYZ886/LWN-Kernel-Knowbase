---
title: Improving i_version
url: https://lwn.net/Articles/937247/
date: "July 5, 2023"
category: Filesystems
author: "By Jake Edge July 5, 2023 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
July 5, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

The [`i_version`](<https://elixir.bootlin.com/linux/v6.4.1/source/include/linux/fs.h#L684>) field in [`struct inode`](<https://elixir.bootlin.com/linux/v6.4.1/source/include/linux/fs.h#L608>) is meant to track changes to the data or metadata of a file. There are some [problems](<https://lwn.net/Articles/905931/>) with the way that `i_version` is being handled in the kernel, so Jeff Layton led a filesystem session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) to discuss them and what to do about them. For the most part, there are solutions in the works that will resolve most of the larger issues. 

Layton's motivation for improving the state of `i_version` handling is NFS. Currently, the NFSv3 code watches file/directory timestamps (access time, or atime, and change time, or ctime) to indicate when its cache should be invalidated. But those times are recorded with one-jiffy (1-10ms) resolution; a lot can happen in a jiffy on today's hardware. That can lead to problems with the client thinking that its cache is up-to-date when it really is not. 

[ ![\[Jeff Layton\]](https://static.lwn.net/images/2023/lsfmb-layton-sm.png) ](<https://lwn.net/Articles/937323/>)

For NFSv4, a new "change attribute" was added; it is a 64-bit unsigned quantity that must change any time the ctime would be updated. Originally, it was considered to be an opaque value, but, over time, the advantages of a monotonically increasing value became apparent. In particular, clients can determine whether certain updates have been performed by seeing if the change attribute is higher or lower than the value it expects. 

NFS servers report what kind of change attribute they use; the client can then decide how to treat the values that it gets. Right now, Linux reports an "undefined" type for its change attribute, but Layton would like to be able to report that the change attribute is monotonically increasing. The inode's `i_version` field can be used for the NFS change attribute; Layton seemed to use the terms `i_version` and "change attribute" mostly interchangeably throughout the session. 

The change attribute must be changed whenever the ctime in the metadata would be changed, as mentioned; some servers can ensure that the attribute change is atomic with respect to the file change that caused it. The Linux server is not able to provide that atomicity, so there is a question of when the attribute should be changed. Right now, for write operations, `i_version` is changed before the file write is visible, which means that someone could race with the server by seeing the new `i_version` value that was caused by a write, then doing a read operation that gives the data currently in the page cache, so the data and `i_version` are out of sync. The client will not update its cache unless another change is made to the file, so this condition can persist for some time. 

Changing `i_version` after the file change becomes visible is "still a little racy", but any synchronization problem should not last long as the client should catch up fairly quickly. Another possibility is to increment the value before and after the file change. 

Layton then looked at the `i_version` field in a bit more detail. It is an unsigned 64-bit value stored in `struct inode` that comes in two flavors. The first is "server managed" and is used by network-filesystem clients (e.g. NFS or CephFS clients); the value stored in the local inode is whatever value the server has. Local filesystems use a kernel-managed `i_version`; the kernel increments the value when it updates the metadata timestamps for the file. There is VFS infrastructure that filesystems can opt into using the `SB_I_VERSION` flag; so far, ext4, Btrfs, XFS, and tmpfs are using it. 

The kernel-managed `i_version` is the more interesting one, he said; the infrastructure is already enabled in the four filesystems he mentioned and GFS2 plans to enable it, though work has not yet started. Originally, it was a simple counter that got incremented whenever ctime was updated, but that turned out to be costly for ext4 and XFS because each increment needed to be logged to disk even if nothing else changed. 

Around 2018, there was a switch to a new scheme that sacrificed the low-order bit of the counter for a "queried" flag. If the `i_version` value was queried, the bit was set. When it was time to increment the counter, the flag was checked; if the flag was set, the counter needed to be incremented, but if not, the counter could stay at the same value. That change allowed the filesystem developers to regain the performance that was lost in the original scheme. 

But there are still some problems, Layton said. A while back, he noticed that XFS and ext4 were updating `i_version` based on atime updates, but it does not make sense to invalidate the cache for a file because someone simply read it; that has been fixed in ext4, but XFS uses `i_version` for some other things so some other solution must be found for that. 

For file writes, `i_version` is being updated before copying the changes to the page cache, which can lead to the problems he described earlier. There is the potential for losing updates due to crashes because NFSD does not wait for the updated value to be written by the filesystem before it starts presenting it to clients. That could lead to a client with an `i_version` and file data that correspond to the data lost in the crash, while another client has the same value but different data. NFSD mitigates this problem by using the ctime value to differentiate the two file versions for filesystems that need it. In addition, the `i_version` behavior is difficult to test because there is no way to query it from user space without changing its behavior (i.e. setting the "queried" bit). 

Generally, `i_version` is updated alongside the ctime update. For directories, that means it is updated after the operation completes, but for writes to files, it is generally done before the data is copied. One way to be more consistent would be to separate the ctime and `i_version` updates; there is resistance to changing when ctime updates are done, with good reason, but `i_version` updates could be moved to after the operation is performed. There are still some possible races, but that would be better. 

Another possibility is to bump `i_version` both before and after the operation. In nearly all cases, it is a no-op anyway because of the queried bit. Meanwhile, though, XFS does not need any changes of this sort because it serializes buffered reads and writes. Ext4, Btrfs, and tmpfs, though, should probably also increment `i_version` after the operation completes. 

Layton said that an idea from Dave Chinner for a multi-grain timestamp for ctime could be used. Chinner suggested that NFSD use ctime for its change attribute (instead of `i_version`), but that the updates to it be done at jiffy resolution except when the value has been queried. At that point, the ctime value gets updated with a fine-grained (much higher than jiffy-resolution) timestamp. Layton has [posted some patches](<https://lwn.net/Articles/930795/>) to implement the idea; there are some test failures, but they are due to faulty tests, he said. 

For the future, though, he thinks it would be quite useful to expose the change attribute to user space, perhaps via [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) (e.g. with a `STATX_CHANGE_COOKIE` type). It would allow creating a "gated write" that stores the attribute, reads the data and modifies it in memory, then only writes it if the attribute value has not changed; otherwise, it tries again starting with retrieving the change attribute. That is similar to something he did in CephFS a ways back and it provides consistency without requiring locks. 

Ted Ts'o thought that decoupling the cookie from `i_version`, so that it was only an in-memory value, might be better, but Layton said that the same value needs to be used by NFSD for crash resilience, so it needs to be on the disk as well. Christian Brauner wanted to make sure that there were clearly defined, consistent semantics for the cookie value if were to be added to `statx()`. He complained that the meaning of the `f_fsid` value for the [`statfs()` system call](<https://man7.org/linux/man-pages/man2/statfs.2.html>) is amorphous; "nobody knows what it is supposed to mean". Layton agreed that it will be important to spell that out. 

There was some discussion of the differences between change attributes as defined for NFS versus the ones that the Andrew filesystem (AFS) uses; the latter is only for changes to the data, so metadata changes do not update its change attribute. Meanwhile, though, NFS has to handle the case of local modifications of the filesystem, while AFS does not; NFSD itself cannot fully manage the updates to the change attribute because the value needs to be updated when local modifications are made. The NFS server will not even know that the modification has occurred. 

In the end, it was generally agreed that the multi-grain timestamp approach for ctime should be pursued. It will give user space sufficient information, so the change cookie for `statx()` likely will not be needed. Layton said he would be working on adding that functionality, but that he needs to fix a number of tests as part of that work.
