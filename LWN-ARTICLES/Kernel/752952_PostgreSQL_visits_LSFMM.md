---
title: PostgreSQL visits LSFMM
url: https://lwn.net/Articles/752952/
date: "May 1, 2018"
category: "Block layer-Error handling"
author: "By Jake Edge May 1, 2018 LSFMM"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
May 1, 2018

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2018/>)

The recent [`fsync()` woes](<https://lwn.net/Articles/752063/>) experienced by PostgreSQL led to a [session](<https://lwn.net/Articles/752613/>) on the first day (April 23) of the 2018 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM). Those problems also led to a second-day session with PostgreSQL developer Andres Freund who gave an overview of how PostgreSQL does I/O and where that ran aground on some assumptions that had been made. The session led to a fair amount of discussion with the filesystem-track developers; real solutions seem to be in the offing. 

PostgreSQL is process-based; there are no threads used, Freund said. It does write-ahead logging (WAL) for durability and replication. That means it logs data before it is marked dirty and the log is flushed before the dirty data is written. Checkpointing is done in the background with writes that are throttled as needed. In general, all data I/O is buffered, though the WAL can use direct I/O. 

[ ![\[Andres Freund\]](https://static.lwn.net/images/2018/lsf-freund-sm.jpg) ](<https://lwn.net/Articles/753183/>)

There is a per-process file descriptor cache with a size limited by the kernel configuration and `ulimit`, so file descriptors are closed if there are not enough available. On Linux, the dirty data is forced to storage by an explicit [`sync_file_range()`](<http://man7.org/linux/man-pages/man2/sync_file_range.2.html>) with the `SYNC_FILE_RANGE_WRITE` flag. Writes come from several sources: the checkpointer writes sorted pages, the background writer does largely random writes, and the backends do random writes. The latter two are pre-cleaning or cleaning various pages, Freund said. 

After that brief overview of PostgreSQL I/O, he moved into the issues the project has run into with `fsync()`. To start with, the guarantees made by Linux (or POSIX) with respect to `fsync()` behavior are not well documented. One wrong assumption that was made was that retrying an `fsync()` will fail if the underlying problem has not been fixed. Other operating systems (FreeBSD and Solaris, at least) do have that behavior. Handling that difference is fairly straightforward, he said. 

A bigger problem is that it was assumed that `fsync()` would return an error if there was a writeback failure, which is not necessarily true for Linux. That was never reliable, but it got a bit worse for PostgreSQL after the [introduction of `errseq_t`](<https://lwn.net/Articles/724307/>), which is what led to the recent fallout. Matthew Wilcox has a [patch](<https://lkml.org/lkml/2018/4/23/994>) that makes things better, but still provides no guarantee. In order for all of that to work, however, PostgreSQL would need to have at least one file descriptor that stays open from the earliest write, which is not possible at the moment. It is not just PostgreSQL that is affected, Freund said, backup tools, rsync, and others are impacted as well. 

Amir Goldstein asked if there were tests to reproduce the problems that PostgreSQL is seeing. Freund said the project has some, but that they need to improve. A crash framework that uses device-mapper failure injection is under development, he said. Ted Ts'o said that xfstests has ways to do that kind of testing as well, so PostgreSQL should look into that for ideas and code. 

Freund said that some have suggested that using direct I/O (DIO) would be a solution for the database system. There are architectural issues that make DIO perform poorly for PostgreSQL, but the project is working on them. In addition, DIO is only going to be useful for well-tuned databases—many installed PostgreSQL databases are not. 

One of the possible solutions that PostgreSQL has investigated is to pass file descriptors to the checkpointer, which is what will be calling `fsync()`. One of the problems with that is to figure out which descriptor for the file is the oldest. Wilcox asked whether the descriptors that need to be closed could be synced before they are closed. That would be too slow, Freund said, since there are potentially hundreds or thousands of file descriptors that would be affected. 

David Howells asked if a new option to [`fadvise64()`](<http://man7.org/linux/man-pages/man2/fadvise64.2.html>) that returns the error count would be helpful. Freund said that would be one of the best solutions to the problems PostgreSQL is having. A per-filesystem error count would be sufficient; the database would then figure out what it needed to do from that. 

Jan Kara said that, for the near term, the plan should be to get Wilcox's patch merged and to work up a patch to keep inodes with errors in memory, as had been discussed the day before. If those inodes are not evicted, the errors can be reliably reported. Since then, the patch from Wilcox [has been merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b4678df184b314a2bd47d2329feca2c2534aa12b>), with the stable kernel team being copied, so it should appear in stable kernels too before long. 

There was talk of some way to monitor the kernel log for I/O errors (or to get that kind of information reported via netlink sockets, as Google does). That would work, Freund said, but it is overkill. In the end, PostgreSQL does not really care what the error is, just that it occurred. In addition, a fix that doesn't require rsync, tar, and others to change in order to receive errors that way is much preferred. 

In closing, Freund asked for some documentation that would tell application developers what needs to be done in order to durably write their data to disk. Dave Chinner claimed that was "asking too much", to a fair amount of laughter. On the other hand, though, no one really stepped up to say they planned to write said documentation either. Freund did [post](<https://lwn.net/Articles/753184/>) a summary of what he learned at LSFMM to the pgsql-hackers mailing list.
