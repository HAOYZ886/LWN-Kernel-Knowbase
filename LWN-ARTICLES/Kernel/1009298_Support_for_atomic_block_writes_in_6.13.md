---
title: Support for atomic block writes in 6.13
url: https://lwn.net/Articles/1009298/
date: "February 20, 2025"
category: "Atomic IO operations; Block layer-Atomic operations"
author: "February 20, 2025 This article was contributed by Ritesh Harjani and Ojaswin Mujoo"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 20, 2025

This article was contributed by Ritesh Harjani and Ojaswin Mujoo

Atomic block writes, which [have been discussed here](<https://lwn.net/Kernel/Index/#Atomic_IO_operations>) a few times in the past, are block operations that either complete fully or do not occur at all, ensuring data consistency and preventing partial (or "torn") writes. This means the disk will, at all times, contain either the complete new data from the atomic write operation or the complete old data from a previous write. It will never have a mix of both the old and the new data, even if a power failure occurs during an ongoing atomic write operation. Atomic writes have been of interest to many Linux users, particularly database developers, as this feature can provide significant performance improvements. 

The Linux 6.13 merge window included a pull request from VFS maintainer Christian Brauner titled "[vfs untorn writes](<https://lwn.net/ml/all/20241115-vfs-untorn-writes-7229611aeacc@brauner/>)", which added the initial atomic-write capability to the kernel. In this article, we will briefly cover what these atomic writes are, why they are important in database world, and what is currently supported in the 6.13 kernel. 

To support atomic writes, changes were required across various layers of the Linux I/O stack. At the VFS level, an interface was introduced to allow applications to request atomic write I/O, along with enhancements to [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) to query atomic-write capabilities. Filesystems had to ensure that physical extent allocations were aligned to the underlying device's constraints, preventing extents from crossing atomic-write boundaries. For example, NVMe namespaces may define atomic boundaries; writes that straddle these boundaries will lose atomicity guarantees. 

The block layer [was updated](<https://lwn.net/ml/all/20240620125359.2684798-1-john.g.garry@oracle.com/>) to prevent the splitting of in-flight I/O operations for atomic write requests and to propagate the device constraints for atomic writes to higher layers. Device drivers were also modified to correctly queue atomic write requests to the hardware. Finally, the underlying disk itself must support atomic writes at the hardware level. Both NVMe and SCSI provide this feature, but in different ways; NVMe implicitly supports atomic writes for operations that remain within specified constraints, but SCSI requires a special command to ensure atomicity. 

#### Why do databases care?

A common practice in databases is to perform disk I/O in fixed-size chunks, with 8KB and 16KB being popular I/O sizes. Databases also, however, maintain a journal that records enough information to enable recovery from a possible write error. The idea is that, if the write of new data fails, the database can take the old data present on disk as a starting point and use the information in the journal to reconstruct the new data. However, this technique is based on the assumption that the old data on disk is still consistent after the error, which may not hold if a write operation has been torn. 

Tearing may happen if the I/O stack doesn't guarantee atomicity. The multi-KB write issued by the database could be split by the kernel (or the hardware) into multiple, smaller write operations. This splitting could result in a mix of old and new data being on disk after a write failure, thus leading to inconsistent on-disk data which can't be used for recovery. 

To work around this possibility, databases employ an additional technique called "double write". In this approach, they first write a copy of the older data to a temporary storage area on disk and ensure that the operation completes successfully before writing to the actual on-disk tables. In case of an error in that second write operation, databases can recover by performing a journal replay on the saved copy of the older data, thus ensuring an accurate data recovery. But, as we can guess, these double writes come at a significant performance cost, especially for write-heavy workloads. This is the reason atomicity is sought after by databases; if the I/O stack can ensure that the chunks will never be torn, then databases can safely disable double writes without risking data corruption and, hence, can get that lost performance back. 

#### Current state in Linux

As discussed during [LSFMM+BPF 2024](<https://lwn.net/Articles/974578/>), some cloud vendors might already advertise atomic-write support using the ext4 filesystem with [bigalloc](<https://docs.kernel.org/filesystems/ext4/overview.html#bigalloc>), a feature that enables cluster-based allocation instead of per-block allocation. This helps to properly allocate aligned physical blocks (clusters) for atomic write operations. However, claiming to support atomic writes after auditing code to convince oneself that the kernel doesn't split a write request is one thing, while properly integrating atomic-write support with a well-defined user interface that guarantees atomicity is another. 

With the Linux 6.13 release, the kernel provides a user interface for atomic writes using direct I/O. Although it has certain limitations (discussed later in this article), this marks an important step toward enabling database developers to explore these interfaces. 

A block device's atomic-write capabilities are stored in [`struct queue_limits`](<https://elixir.bootlin.com/linux/v6.13.2/source/include/linux/blkdev.h#L358>). These limits are exposed to user space via the sysfs interface at `/sys/block/<device>/queue/atomic_*`. The files `atomic_write_unit_min` and `atomic_write_unit_max` indicate the minimum and maximum number of bytes that can be written atomically. If these values are nonzero, the underlying block device supports atomic writes. However, hardware support alone is not sufficient; as mentioned earlier, the entire software stack, including the filesystem, block layer, and VFS, must also support atomic writes. 

#### How to use the atomic-write feature

Currently, atomic-write support is only enabled for a single filesystem block. Multi-block support is [under development](<https://lwn.net/ml/all/20241204154344.3034362-1-john.g.garry@oracle.com/>), but those operations bring some more constraints that are still being discussed in the community. To utilize the current atomic write feature in Linux 6.13, the filesystem must be formatted with a block size that is suitable for the application's needs. A good choice is often 16KB. 

Note, though, that ext4 does not support filesystem block sizes greater than the system's page size, so, on systems with 4KB page size (such as x86), ext4 cannot use a block size of 16KB and, thus, cannot support atomic write operations of that size. On the other hand, XFS recently got [large block size support](<https://lwn.net/Articles/933437/>), allowing it to handle block sizes greater than page size. Note also that there is no problem with ext4 or XFS if the page size of the system itself is either 16KB or 64KB (such as on arm64 or powerpc64 systems), as both filesystems can handle block sizes less than or equal to the system's page size. 

The following steps show how to make use of the atomic-write feature: 

  1. First create a filesystem (ext4 or xfs) with a suitable block size based on the atomic-write unit supported by the underlying block device. For example: 

```
mkfs.ext4 -b 16K /dev/sdd
             mkfs.xfs -bsize=16K /dev/sdd
```

  2. Next, use the `statx()` system call to confirm whether atomic writes are supported on a file by the underlying filesystem. Unlike checking the block device sysfs path, which only indicates whether the underlying disk supports atomic writes, `statx()` allows the application to query whether it is possible to request an atomic write operation on a file and determine the supported unit size, which also ensures that the entire I/O stack supports atomic writes. 

To facilitate atomic writes, `statx()` now exposes the following fields when the `STATX_WRITE_ATOMIC` flag is passed: 
     * `stx_atomic_write_unit_min`: Minimum size of an atomic write request.
     * `stx_atomic_write_unit_max`: Maximum size of an atomic write request.
     * `stx_atomic_write_segments_max`: Upper limit for segments — the number of separate memory buffers that can be gathered into a write operation (e.g., the `iovcnt` parameter for `IOV_ITER`). Currently, this is always set to one.
     * The `STATX_ATTR_WRITE_ATOMIC` flag in `statx->attributes` is set if atomic writes are supported.

An example `statx()` snippet would look like the following: 

```
statx(AT_FDCWD, file_path, 0, STATX_BASIC_STATS | STATX_WRITE_ATOMIC, &stat_buf);
    
        printf("Atomic write Min: %d\n", stat_buf.stx_atomic_write_unit_min);
        printf("Atomic write Max: %d\n", stat_buf.stx_atomic_write_unit_max);
```

  3. Finally, to perform an atomic write, open the file in `O_DIRECT` mode and issue a [`pwritev2()`](<https://man7.org/linux/man-pages/man2/readv.2.html>) system call with the `RWF_ATOMIC` flag set. Ensure that the total length of the write is a power of two that falls between `atomic_write_unit_min` and `atomic_write_unit_max`, and that the write starts at a naturally aligned offset in the file with respect to the total length of the write. 

Currently, `pwritev2()` with `RWF_ATOMIC` supports only a single `iovec` and is limited to a single filesystem block write. This means that filesystems, when queried via `statx()`, report both the minimum and maximum atomic-write unit as a single filesystem block (e.g., 16KB in the example above). 

#### The future

Kernel developers have implemented initial support for direct I/O atomic writes that are limited to a single filesystem block. However, there is an ongoing work which aims to extend the support to multi-block atomic writes for both the [ext4](<https://lwn.net/ml/all/f5bd55d32031b49bdd9e2c6d073787d1ac4b6d78.1729825985.git.ritesh.list@gmail.com/>) and [XFS](<https://lwn.net/ml/all/20241204154344.3034362-1-john.g.garry@oracle.com/>) filesystems. Despite its limitations, this feature provides a foundation for those interested in atomic-write support in Linux. This also presents an opportunity for users, such as database developers, to start exploring and experimenting with this feature. One can still collaborate with the community to enhance this feature, as it is still under active [discussion](<https://lwn.net/ml/all/Z6O6g4pCu-pXJql5@bombadil.infradead.org/>) and [development](<https://lwn.net/ml/all/Z5nTaQgLGdD6hSvL@li-dc0c254c-257c-11b2-a85c-98b6c1322444.ibm.com/>).
