---
title: "Toward non-blocking asynchronous I/O"
url: https://lwn.net/Articles/724198/
date: "May 30, 2017"
category: Asynchronous IO
author: "By Jonathan Corbet May 30, 2017"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
May 30, 2017

The Linux asynchronous I/O (AIO) layer tends to have many critics and few defenders, but most people at least expect it to actually be asynchronous. In truth, an AIO operation can block in the kernel for a number of reasons, making AIO difficult to use in situations where the calling thread truly cannot afford to block. A longstanding patch set aiming to improve this situation would appear to be nearing completion, but it is more of a step in the right direction than a true solution to the problem. 

To perform AIO, a program must set up an I/O context with [`io_setup()`](<http://man7.org/linux/man-pages/man2/io_setup.2.html>), fill in one or more [`iocb` structures](<http://elixir.free-electrons.com/linux/latest/source/include/uapi/linux/aio_abi.h#L73>) describing the operation(s) to be performed, then submit those structures with [`io_submit()`](<http://man7.org/linux/man-pages/man2/io_submit.2.html>). A call to [`io_getevents()`](<http://man7.org/linux/man-pages/man2/io_getevents.2.html>) can be made to learn about the status of outstanding I/O operations and, optionally, wait for them. All of those system calls should, with the exception of the last, be non-blocking. In the real world, things are more complicated. Memory allocations or lock contention can cause any AIO operation to block before it starts to move any data at all. And, even in the best-supported case (direct file I/O), the operation itself can block in a number of places. 

The [no-wait AIO patch set](<https://lwn.net/Articles/722784/>) from Goldwyn Rodrigues seeks to improve this situation in a number of ways. It does not make AIO any more asynchronous, but it will cause AIO operations to fail with `EAGAIN` errors rather than block in a number of situations. If a program is prepared for such errors, it can opportunistically try to submit I/O in its main thread; it will then only need to fall back to a separate submission thread in cases where the operation would block. 

If a program is designed to use no-wait AIO, it must indicate the fact by setting the new `IOCB_RW_FLAG_NOWAIT` flag in the `iocb` structure. That structure has a field (`aio_flags`) that is meant to hold just this type of flag, but there is a problem: the kernel does not currently check for unknown flags in that field. That makes it impossible to add a new flag, since a calling program can never know whether the kernel it is running on supports that flag or not. Fortunately, that structure contains a couple of reserved fields that _are_ checked in current kernels; the field formerly known as `aio_reserved1` is changed to `aio_rw_flags` in this patch set and used for the new flag. 

One of the places where an I/O request can block is if the operation will trigger a writeback operation; in that case, the request will be held up until the writeback completes. This wait happens early in the submission process; in particular, it can happen before `io_submit()` completes its work and returns. Setting `IOCB_RW_FLAG_NOWAIT` will cause submission to fail with `EAGAIN` in this case. 

Another common blocking point is I/O submission at the block level, where, in particular, a request can be stalled because the underlying block device is too busy. Avoiding that involves the creation of a new `REQ_NOWAIT` flag that can be set in the BIO structure used to describe block I/O requests. When that flag is present, I/O submission will, once again, fail with an `EAGAIN` error rather than block waiting for the level of block-device congestion to fall. 

Support is also needed at the filesystem level; each filesystem has its own places where execution can block on the way to submitting a request. The patch set includes support for Btrfs, ext4, and XFS. In each case, situations like the inability to obtain a lock on the relevant inode will cause a request to fail. 

All of this work can make AIO better, but only for a limited set of use cases. It only improves direct I/O, for example. Buffered I/O, which has always been a sort of second-class citizen in the AIO layer, is unchanged; there are simply too many places where things can block to try to deal with them all. Similarly, there is no support for network filesystems or for filesystems on MD or LVM volumes — though Rodrigues plans to fill some of those gaps at some future point. 

In other words, AIO seems likely to remain useful only for the handful of applications that perform direct I/O to files. There have been a number of attempts to improve the situation in the past, including [fibrils](<https://lwn.net/Articles/219954/>), [threadlets](<https://lwn.net/Articles/223899/>), [syslets](<https://lwn.net/Articles/221913/>), [acall](<https://lwn.net/Articles/316806/>), and [an AIO reimplementation based on kernel threads](<https://lwn.net/Articles/671649/>) done by the original AIO author. None of those have ever reached the point of being seriously considered for merging into the mainline, though. There are a lot of tricky details to be handled to implement a complete solution, and nobody has ever found the goal to be important enough to justify the considerable work required to come up with a better solution to the problem. So the kernel will almost certainly continue to crawl forward with incremental improvements to AIO.
