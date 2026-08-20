---
title: Atomic writes without tears
url: https://lwn.net/Articles/974578/
date: "May 24, 2024"
category: "Atomic IO operations; Block layer-Atomic operations"
author: "By Jake Edge May 24, 2024 LSFMM+BPF"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jake Edge**  
May 24, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

John Garry and Ted Ts'o led a discussion about supporting atomic writes for buffered I/O, without any torn (or partial) writes to the device, at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>). It is something of a continuation of a [discussion at last year's summit](<https://lwn.net/Articles/933015/>). The goal is to help PostgreSQL, which writes its data using 16KB buffered I/O; it currently has to do a lot of extra work to ensure that its data is safe on disk. A promise of non-torn, 16KB buffered writes would allow the database to avoid doing double writes. 

[ ![\[John Garry\]](https://static.lwn.net/images/2024/lsfmb-garry-sm.png) ](<https://lwn.net/Articles/974952/>)

Garry began by going over the problem being solved; databases generally write their data in chunks larger than the block size of the block device (which is generally the same as the page size of the system, 4KB). MySQL and PostgreSQL both use larger chunks, up to 16KB. They need to ensure the persistence of these chunks, in full, in order to maintain an uncorrupted database. MySQL uses direct I/O, which is generally able to ensure that 16KB is either fully written, or not written at all, on today's storage hardware. 

The kernel does not guarantee atomic 16KB writes, even for direct I/O, however. So Garry has come up with a [patch set for supporting atomic block writes](<https://lwn.net/ml/linux-kernel/20240326133813.3224593-1-john.g.garry@oracle.com/>) (as well as one to [add the feature to XFS](<https://lwn.net/ml/linux-kernel/20240429174746.2132161-1-john.g.garry@oracle.com/>)). Later in the session he said that there is ongoing work to support the feature in ext4; he has also posted an [RFC patch set for buffered atomic writes](<https://lwn.net/ml/linux-kernel/20240422143923.3927601-1-john.g.garry%40oracle.com/>). 

In his patches, there is a new `RWF_ATOMIC` flag for [`pwritev2()`](<https://man7.org/linux/man-pages/man2/writev.2.html>) that requests torn-write protection for the write, he said; there is a corresponding flag for io_uring as well. The `RWF_SYNC` flag is still needed to guarantee persistence, though. The [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) system call can be used to query some new fields to determine the minimum and maximum atomic sizes supported, as well as the maximum number of atomic segments allowed for a given write operation. All of those values are dependent on the underlying filesystem, block-layer, and storage-device limitations. 

In order to make a call using `RWF_ATOMIC`, the total length must be a power of two, between the minimum and maximum, be "naturally aligned", and the write must have an `iov_count` no more than the maximum number of segments. Damien Le Moal asked whether this feature required hardware support in order to ensure persistence with `RWF_SYNC`; Garry said that it did. Hannes Reinecke asked what was meant by "naturally aligned offset"; Garry said that it means aligning an 8KB write on an 8KB boundary, 16KB on 16KB, and so on. 

#### Hardware

Both SCSI and NVMe support torn-write protection, Garry continued, but they do it differently. NVMe implicitly does atomic writes; there is no dedicated command to request them. Devices have a limit and if the write is less than that, and does not cross the device-specific atomic-write boundary, if any, it will be written atomically. SCSI has a separate command and more constraints; unlike NVMe, though, atomic writes that do not meet the requirements are rejected. Reinecke asked whether it made sense to merge BIOs in an atomic-write request; Garry said that they will not be merged if the combination cannot be done as an atomic write. 

The XFS support adds a new `FORCEALIGN` inode flag that can be set via an [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>) command, which forces the file to be aligned on certain boundaries. That can be used to ensure that file extents are 16KB-aligned, for example, on a per-inode basis. 

[ ![\[Ted Ts'o\]](https://static.lwn.net/images/2024/lsfmb-tso-sm.png) ](<https://lwn.net/Articles/974953/>)

Ts'o said that the cloud vendors are already advertising torn-write protection for MySQL; they are using ext4 with particular settings on devices that can provide that protection. But there are "lots of sharp edges"; the vendors have to audit the code paths and hope that a kernel update does not break them in some way, since the kernel makes no guarantees. The feature can provide a 60-100% improvement in database performance, he said, because MySQL can avoid doing a double write, which makes it attractive. 

He said that the database developers want write requests that are only ever torn at 16KB boundaries. With the atomic-write support, kernel developers are trying to do better than that, but it is important to ensure that the huge performance gain does not get diminished or lost in the process. For example, the database might send a contiguous 256KB write, but the only thing the developers are looking for is to know if the I/O fails for some reason and that the failure can only happen at 16KB boundaries. It is important to remember that the database developers want "untorn writes"; guaranteeing more with atomic writes is fine, Ts'o said, as long as it comes for free. 

Dave Chinner said that XFS added atomic writes ten years ago, which doubled MySQL performance, so the effects of this kind of change have been known for a long time. Garry said that the term "atomic" is used for the feature, because that is what the hardware vendors call it, but that it is providing the "untorn writes" that database developers want. Matthew Wilcox noted that NVMe is specified to have 16KB tear boundaries; he wondered if the SCSI vendors could be convinced into doing something similar. But Martin Petersen wanted to know what problems there were with the current SCSI semantics; there are differences with NVMe, but he is unsure why they are a problem. It turned out that the currently proposed implementation for atomic writes does not need to use everything that SCSI provides, so it is not clear whether there are any actual deficiencies. 

#### Buffered I/O

Ts'o said that the buffered I/O piece is where this all gets interesting. The proposed API works great for direct I/O, because right after the call to `pwritev2()`, the I/O is actually done to the device. For buffered I/O, that is not the case, since everything is going through the page cache, which means that the write may not actually happen for "30 seconds or until user space calls [`fdatasync()`](<https://man7.org/linux/man-pages/man3/fdatasync.3p.html>)". The reason for caring about buffered I/O is because PostgreSQL uses it; depending on who you talk to, it will be three to six years before the database can switch to using direct I/O. 

Part of the problem is that using the proposed API means keeping track of which writes were done using the atomic flag. If a 64KB write is done, then a 16KB write, both with the flag, they need to be tracked separately. There has also been talk of a hybrid mode, he said, where a non-atomic-aware application can also write to the file in a non-aligned way such that "things don't blow up". The problem has become over-constrained; `pwritev2()` is fine for direct I/O, but does not really fit with what the database developers are asking for in the buffered I/O case. 

There are multiple ways to create an interface for buffered I/O, Ts'o said. It could be an inode-based write granularity, set with something like the XFS `FORCEALIGN` flag, an `ioctl()` command, or [`fcntl()`](<https://man7.org/linux/man-pages/man2/fcntl.2.html>) operation. Then there is a question of whether to stick with the `pwritev2()` interface, which is more powerful than what is needed, or, perhaps, to require that the application using buffered I/O only do atomic writes at the granularity specified. That would mean that the kernel does not have to track various in-flight atomic-write sizes. Another way to do that might be to require that the size of the folio used for the write specifies the granularity. 

An attendee said that with buffered I/O, there is no way for the application to get an error if the write fails. Ts'o said that any error would come when `fdatasync()` is called, which the attendee called "buffered in name only". But Ts'o said that it is how PostgreSQL works today; it does multiple buffered writes, then calls `fdatasync()` to make that data persistent and to detect if there are any problems. The developers understand that model and it is the documented POSIX interface. 

Jan Kara suggested that instead of tracking different sizes of atomic writes, a single size could be tracked; if another write comes in with a different size, the earlier writes could be flushed out. In his RFC, Garry said, the granularity was effectively set by the `FORCEALIGN` size. 

There was some discussion of the SCSI semantics with respect to whether reads were synchronized with writes, and whether that means an atomic read operation is also needed. The answer seemed to be that no atomic read was needed. But, because SCSI has separate write commands for atomic versus non-atomic, there does need to be some kind of indication from user space about the kind of I/O it expects. That could be done with a flag on the inode or folio. 

Chinner suggested that the page-cache code could interpret these flags and implement writethrough for writes of that sort. It could be implemented using the direct I/O code, so that those kinds of writes are not truly buffered. But Garry said that the tricky piece is handling a mixture of atomic and non-atomic writes on the same folio. 

The only reason an application would be using atomic writes, though, is for performance, Chinner said. Trying to support both types of writes, including non-aligned writes, does not make any sense. It comes down to a question of whether it is an error to mix the two types of writing for the same file, Ts'o said, or if there is a call to `pwritev2()` with the wrong alignment; there is a need to clearly define what the semantics are. 

Kara asked about the impact of these changes on the database code. Ts'o said that he believes the PostgreSQL developers are looking forward to a solution, so they are willing to make the required changes when the time comes. They are likely to backport those changes to earlier versions of PostgreSQL as well. Wilcox said that probably did not matter, because the older versions of PostgreSQL are running on neolithic kernels. Ts'o said that is not necessarily the case since some customers are willing to upgrade their kernel, but require sticking with the older database system. 

The discussion trailed off at that point so any further progress will presumably come on the mailing list.
