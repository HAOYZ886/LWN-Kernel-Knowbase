---
title: Handling the NFS change attribute
url: https://lwn.net/Articles/975863/
date: "June 4, 2024"
category: Filesystems
author: "By Jake Edge June 4, 2024 LSFMM+BPF"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
June 4, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

The saga of the `i_version` field for inodes, which tracks the occurrence of changes to the data or metadata of a file, continued in a discussion at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). In a session led by Jeff Layton, who has been doing a lot the work on [changing the semantics and functioning of `i_version`](<https://lwn.net/Articles/905931/>) over the years, he updated attendees on the status of the [effort](<https://lwn.net/Articles/946394/>) since a [session at last year's summit](<https://lwn.net/Articles/937247/>). His summary was that things are "pretty much where we started last year", but the discussion this time pointed to some possible ways forward. 

#### Granularity

The problem is that the granularity of the timestamps used by Linux (generally 1-10ms) is not sufficient to actually record all of the changes that can happen to a file. Multiple writes, for example, could all happen within the same change time (ctime) value. This becomes a problem for NFSv2 and NFSv3 clients, which effectively use the ctime value to decide when to invalidate their cached information; two different versions of a file with the same ctime makes for a mess. 

[ ![\[Jeff Layton\]](https://static.lwn.net/images/2024/lsfmb-layton-sm.png) ](<https://lwn.net/Articles/976353/>)

NFSv4 added a "change attribute", which is a 64-bit value that is guaranteed to change any time that the ctime would change (effectively), he said; it does not get updated when the access time (atime) changes, because the caches should not be invalidated when files are read. NFSv4.2 added the ability for the server to indicate what kind of change-attribute information it is providing, which may allow clients to make better caching decisions. For example, if it is reported as monotonically increasing, clients can ignore updates with lower values; only change attributes that are higher than the value in the cache are valid at that point. 

Most Linux filesystems track the change-attribute information as the `i_version` of the file's inode. But different filesystems handle the attribute somewhat differently. In particular, XFS has its own attribute that does not follow the same semantics as the others—it is incremented for atime updates. So, if atime updates are turned on, the client caches are invalidated incorrectly; even the [relative-atime option](<https://lwn.net/Articles/244829/>) can cause some incorrect cache invalidations. 

In the past, XFS developers have been reluctant to add space in the on-disk inode for a change attribute that works in the expected way. He spoke with Darrick Wong earlier in the day, though, and got the sense that perhaps that reluctance might be diminishing. Bcachefs still needs to implement support for the attribute, but the space for it in the inode has been reserved, Layton said. 

Another problem is that, on a write, the attribute value is typically incremented (and the timestamps are updated) before copying the data to the page cache. A read that comes in between the updates and the copy will associate the wrong state of the file with the data that is read. That problem can then persist for a long time in the client—until the file is updated again. 

Moving the updates after the copy still leaves a window for incorrect information on the client, but it should resolve itself quickly. Kent Overstreet asked if the race condition can truly be eliminated. Layton said that moving the updates helps, but does not get rid of the race; clients may have the new data associated with the old attribute value, but they should get the new attribute value soon and invalidate their cache. 

The change attributes are not stored on disk immediately, so server crashes can lead to problems where different file states end up with the same attribute values. Amir Goldstein mentioned some patches he is working on that will use [sleepable RCU](<https://lwn.net/Articles/202847/>) to protect the write operation, so that values can be updated in memory, but will not be written to disk until the full operation has completed. Layton said that he would look to see if the patch set could be used to help with this problem. 

The crash-loss problem can be remedied by using the ctime value combined with the change attribute, which means there can only be a problem if there is a crash and a clock rollback on the server, "which is all pretty unlikely". One thing that makes it hard to test these kinds of problems is that the change attribute is not accessible from user space, so he would like to expose it in some fashion. 

#### Multi-grain

Last year, Dave Chinner had an idea for [multi-grain timestamps](<https://lwn.net/ml/linux-kernel/20230424151104.175456-1-jlayton@kernel.org/>) that was implemented and, briefly, merged. It turned out that there was a problem where an operation with a fine-grained timestamp and another with a coarse-grained one could be seen as happening in the wrong order, Layton said. That breaks "some little-known tools like make and rsync", so the change was backed out. He thinks the problem could be fixed by using the fine-grained timestamp as the floor for coarse-grained updates from that point on, but he got the impression that Linus Torvalds and Christian Brauner were tired of him pushing it. It could be resurrected; Brauner pointed out that his objections were only meant to apply to the merge window that was active at the time, so Layton may pick that work back up. 

Another alternative would be to use some "extra" low bits in the ctime field for a counter that could be bumped every time there is more than one operation in a single timer tick. The timestamps could be shifted appropriately when they were reported to user space and used in full as change attributes. That would require changing all filesystems, though, so that there were not different granularities of timestamps being reported on a given system, Layton said. 

He then went through the order of operations for updating timestamps and `i_version`. There is no locking done for queries of `i_version`; that means that as soon as the value is updated, which is currently done before the copy to the page cache, it can be read. Normally, ctime is updated at the same time as `i_version`, before the write operation; for directories, though, those updates are done after the operation because there is a lock being held. 

In truth, `i_version` is only updated if it has been queried since the last time it was changed, so the increment is often a no-op. One way to handle the race problems, then, might be to increment the value both before and after the operation; the second of those would be a no-op nearly all of the time, so the cost should be minimal. He may experiment with that some. 

Crash resilience is something that he has not yet done sufficient research on, though it has been identified as a potential problem area, he said. He and Jan Kara had an idea for a crash counter that could be tracked by user space; nfsd has a daemon that already tracks some client information where this could be added. It is kind of a "blue sky" idea that would require quite a bit of work, but it would remove the problem of multiple file states with the same change attribute after a crash. That, in turn, would allow the kernel NFS server to report that its change attribute is monotonically increasing, which is advantageous for NFS clients. 

He would like to expose the change attribute via [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) so that user-space programs, such as [NFS-Ganesha](<https://github.com/nfs-ganesha/nfs-ganesha>), could access the value. It will be important to ensure that only filesystems that implement the change attribute with the usual semantics expose it that way, however. That would also allow a feature he has thought about for a long time: a "gated write". The idea would be to fetch the change attribute, then make some changes to the file in memory, and write the file, but only if the change attribute was the same. That would allow synchronizing writes from multiple threads on the same machine, or writes to a network filesystem from multiple machines, without file locking. 

When he asked Layton to lead the session, Goldstein had asked for a "roadmap" to be presented, but Layton said it was "more like a wish list". He wants to add support for the change attribute to bcachefs and to figure out what to do for XFS in that regard. He also wants to move the `i_version` update (and maybe timestamp updates) to after the page-cache copy, or do the double bump that he described. Finally, he wants to figure out what to do about crash resilience. 

Brauner asked about an idea for shrinking inodes by changing the storage of the timestamps. Layton said that came from Torvalds, who pointed out that consecutive [`struct timespec64`](<https://elixir.bootlin.com/linux/v6.9.3/source/include/linux/time64.h#L13>) entries for ctime, mtime, and atime leave alignment gaps. Switching to separate entries for the seconds and nanoseconds, as he did in a [patch](<https://lwn.net/ml/linux-kernel/20240517-amtime-v1-1-7b804ca4be8f@kernel.org/>) posted shortly after the summit, saves eight bytes. There are plans for how to use some of that savings, he said. 

There was some final discussion on the roadmap/wish list, with Ted Ts'o noting that there are no real dependencies between the items, so they could all be worked on in parallel. Wong said that there is actually plenty of room in the XFS inode for a few more counters, but he needed clarification on when the change attribute needed to be updated. It seems like the NFSv4 semantics can be supported in a fairly straightforward fashion, so that piece of puzzle may already be falling into place.
