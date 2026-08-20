---
title: A filesystem for virtualization
url: https://lwn.net/Articles/788333/
date: "May 14, 2019"
category: "Filesystems-Virtualization; Virtualization-virtio"
author: "By Jake Edge May 14, 2019 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
May 14, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

A new filesystem aimed at sharing host filesystems with KVM guests, [virtio-fs](<https://virtio-fs.gitlab.io/>), was the topic of a session led by Miklos Szeredi at the 2019 Linux Storage, Filesystem, and Memory-Management Summit. The existing solution, which is based on the [9P filesystem](<https://lwn.net/Articles/137439/>) from [Plan 9](<https://9p.io/plan9/index.html>), has some shortcomings, he said. Virtio-fs is a prototype that uses the [Filesystem in Userspace](<https://en.wikipedia.org/wiki/Filesystem_in_Userspace>) (FUSE) interface. 

The existing 9P-based filesystem does not provide local filesystem semantics and is "pretty slow", Szeredi said. The FUSE-based virtio-fs ([RFC patches](<https://lwn.net/ml/linux-fsdevel/20181210171318.16998-1-vgoyal@redhat.com/>)) is performing "much better". One of the ideas behind the new filesystem is to share the page cache between the host and guests, so there would be no data duplication for multiple guests accessing the same files from the host filesystem. 

There are still some areas that need work, however. Metadata and the directory entry cache (dcache) cannot be shared, because data structures cannot be shared between the host and guests. There are two ways to handle that. Either there can be a round trip from the guest to the host for each operation to ensure the coherence of the metadata cache and dcache, or the guest can cache that information and somehow revalidate the cache on each operation without going to the host kernel. 

[ ![\[Miklos Szeredi\]](https://static.lwn.net/images/2019/lsf-szeredi-sm.jpg) ](<https://lwn.net/Articles/788336/>)

The question is what the best solution would be, he said. For example, if a file has changed on the host, the modification time is updated and a `stat()` on the guest should indicate that. There have been some discussions on how to get notifications from the host kernel to the guest; the notifications would be propagated via a ring buffer in memory. When the guest caches an inode, it could tell the host that it wants notifications for that inode. When it gets a notification, the guest can revalidate its cache. If the ring buffer overflows for some reason, the guest will need to revalidate all of its caches. 

Amir Goldstein asked if that mechanism could also be used by Samba to implement its own dcache. Trond Myklebust said that what Szeredi was talking about was an asynchronous notification mechanism, while Samba needs something synchronous. The problem with doing synchronous notifications, Szeredi said, is that the guest should not be able to block operations in the host kernel. 

Another topic is POSIX file locking, he said. It is difficult to write a user-space filesystem that allows POSIX locking to work consistently with the host filesystem. The kernel NFS server (knfsd) uses kernel-internal functions to do its locking, but he is not sure what user-space NFS servers do. 

The traditional way to handle that is with a user-space lock manager that takes the standard POSIX locks as needed, Myklebust said. Szeredi asked if it would make sense to add a kernel interface for the kernel-internal locking used by knfsd. Boaz Harrosh said that the [Ganesha NFS server](<https://github.com/nfs-ganesha/nfs-ganesha/wiki>) had a similar problem; it used [open file description locks](<https://www.gnu.org/software/libc/manual/html_node/Open-File-Description-Locks.html>) (OFD locks), which put the lock on the `struct file` so that multiple threads can use the locks successfully, unlike POSIX locks. 

Szeredi said the idea was to have POSIX locks that work across guests and the host. Steve French said that Samba also uses OFD locks, which is what he recommended. They have easier semantics, in part because they don't get lost when the file is closed. It is a solution that was added partly for NFS, he said. Szeredi said that it sounded like the conclusion is that it is not worth it to make a new kernel interface for POSIX locks. 

Another area that needs attention is on the ctime and mtime timestamps stored for files. They record the time of the last metadata update (ctime) and file data update (mtime). If writes to the file are going to a shared page cache, it will cause the timestamps to be updated on the host filesystem, but only sometimes. That could lead to inconsistencies with the guests' metadata caches. 

He is thinking about adding a flag to `open()` to turn off the updating of these timestamps, which would partially solve the problem. XFS already has a flag like this, but it is not exported to user space. That kind of flag may well have security implications, he said. Goldstein said that he thought the flag was added for [Data Management API (DMAPI) support in XFS](<https://en.wikipedia.org/wiki/XFS#DMAPI>) so that it could make changes to files without updating the timestamps. But DMAPI has been deprecated for XFS, which is probably why the flag is not exported. 

The worry about such a flag is that changes can be made to a file's contents without anyone noticing, Myklebust said. That is why it was not added to POSIX, he believes. The solution to the problem is to implement a proper version field that gets exported from the inode.
