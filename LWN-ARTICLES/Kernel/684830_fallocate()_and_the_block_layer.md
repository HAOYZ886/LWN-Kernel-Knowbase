---
title: fallocate() and the block layer
url: https://lwn.net/Articles/684830/
date: "April 27, 2016"
category: fallocate
author: "By Jake Edge April 27, 2016 LSFMM 2016"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
April 27, 2016

* * *

[LSFMM 2016](<https://lwn.net/Articles/lsfmm2016/>)

In a session he dubbed "block device `fallocate()` bikeshedding", Darrick Wong led a discussion on some recent ideas on [moving some functionality](<https://lwn.net/Articles/680708/>) from `ioctl()` commands to a higher level in the stack. The session was in a combined filesystem and storage track session at the 2016 Linux Storage, Filesystem, and Memory-Management Summit. 

There are some block-layer `ioctl()` commands that could be considered as candidates for changing to `fallocate()` flags. For example, he had [proposed](<http://thread.gmane.org/20160302040932.16685.62789.stgit@birch.djwong.org>) a `BLKZEROOUT2` `ioctl()` command to provide a way for user space to access the zeroing facility in the block layer. In the discussion on the mailing list, others, including Linus Torvalds, thought that it made more sense to use the `FALLOC_FL_PUNCH_HOLE` and `FALLOC_FL_ZERO_RANGE` flags to `fallocate()` to do that. 

Wong has [implemented](<http://thread.gmane.org/20160305005556.29738.66782.stgit@birch.djwong.org>) those changes, but wondered about the alignment requirements. His patches currently require that the ranges specified are aligned with the 512-byte logical block. That avoids the complexity of manually zeroing out both ends, while punching out multi-block holes in the middle. Ted Ts'o suggested that, for simplicity, non-aligned ranges should simply get `EINVAL`. 

The conversation then turned to thin provisioning (dm-thinp) in the context of the [out-of-tree `FALLOC_FL_NO_HIDE_STALE` functionality](<http://thread.gmane.org/20160303223952.GE24012@thunk.org>). That flag will cause space to be allocated, but that space will not be zeroed before being made available to user space. So it may return stale data in those blocks, which is generally considered to be a security problem. Dm-thinp needs to allocate space from its pool, but those blocks could still have stale data. By default, dm-thinp will zero out any reads from those regions until they have been written and will zero out the rest of the block if a write is smaller than a block. 

But, more often than not, that zeroing is disabled in dm-thinp by users because it is not needed, since some filesystems (e.g. XFS) already handle getting stale data from the block layer. Wong asked if that "no hide stale" functionality would be useful to in-kernel callers, so that no extra zeroing was done for them. Chris Mason said that many filesystems already expect garbage from the drives, so it would be a surprise to get zeroes. It would make sense to have a way to get zeroes, but Btrfs and others don't really need it. 

A problem with `fallocate()` is that it might take a long time to complete for certain kinds of operations, Wong said. Since using it is supposed to ensure that user space will never receive an `ENOSPC` error when operating on the file, copy-on-write (or `reflink()`-supporting) filesystems need to "unshare" the blocks of a file. For a 1TB file, that effectively means a 1TB copy operation, but not doing so would mean that "the thing that is not supposed to happen, happens". 

So, Wong asked, should there be an `fallocate()` flag for "do expensive operations" to avoid violating users' expectations? But David Howells asked in return: is `fallocate()` supposed to be fast or is it intended to ensure that `ENOSPC` doesn't happen? Christoph Hellwig said that there are already instances where `fallocate()` is not fast. The questions remained unresolved and the session wound down.
