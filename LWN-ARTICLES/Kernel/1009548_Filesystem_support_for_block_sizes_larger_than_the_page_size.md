---
title: Filesystem support  for block sizes larger than the page size
url: https://lwn.net/Articles/1009548/
date: "February 20, 2025"
category: Filesystems
author: "February 20, 2025 This article was contributed by Pankaj Raghav"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 20, 2025

This article was contributed by Pankaj Raghav

The maximum filesystem block size that the kernel can support has always been limited by the host page size for Linux, even if the filesystems could handle larger block sizes. The [large-block-size (LBS) patches](<https://lwn.net/ml/all/20240822135018.1931258-1-kernel@pankajraghav.com/>) that were [merged](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=171754c3808214d4fd8843eab584599a429deb52>) for the 6.12 kernel removed this limitation in XFS, thereby decoupling the page size from the filesystem block size. XFS is the first filesystem to gain this support, with other filesystems likely to add LBS support in the future. In addition, the LBS patches have been used to get the [initial atomic-write support](<https://lwn.net/Articles/974578/>) into XFS. 

LBS is an overloaded term, so it is good to clarify what it means in the context of the kernel. The term LBS is used in this article to refer to places where the filesystem block size is larger than the page size of the system. A filesystem block is the smallest unit of data that the filesystem uses to store file data on the disk. Setting the filesystem block size will only affect the I/O granularity in the data path and will not have any impact on the filesystem metadata. 

#### Long history

The earliest use case for LBS [came from the CD/DVD world](<https://lwn.net/Articles/232757/>), where reads and writes had to be performed in 32KB or 64KB chunks. LBS support was proposed to handle these devices and avoid workarounds at the device-driver level, but it was never merged. Beyond these historical needs, LBS enables testing filesystem block sizes larger than the host page size, allowing developers to verify XFS functionality with 64KB blocks on x86_64 systems that do not support 64KB pages. This is particularly valuable given the increasing adoption of architectures with larger page sizes. 

Another emerging use case for LBS comes from modern high-capacity solid-state storage devices (SSDs). Storage vendors are [increasing their internal mapping unit](<https://www.colfax-intl.com/downloads/intel-achieving-optimal-perf-iu-ssds.pdf>) (commonly called the Indirection Unit or IU) beyond 4KB to support these devices. When I/O operations are not sized for this larger IU, the device must [perform read-modify-write operations](<https://blog.pankajraghav.com/2023/12/18/IU-WAF.html>), increasing the write amplification factor. LBS enables filesystems to match their block size with the device's IU, avoiding these costly operations. 

Although LBS sounds like it has something to do with the block layer, the block-size limit in the kernel actually [comes from the page cache](<https://lwn.net/ml/all/20230308075952.GU2825702@dread.disaster.area/>), not the block layer. The main requirement to get LBS support for filesystems is the ability to track a filesystem block as a single unit in the page cache. Since a block is the smallest unit of data, the page cache should not [partially evict](<https://lwn.net/Articles/945987/>) a single block during writeback. 

There were multiple [attempts](<https://blog.pankajraghav.com/2024/09/04/LBS2.html>) in the past to add LBS support. The [most recent effort](<https://lwn.net/ml/all/20181107063127.3902-1-david@fromorbit.com/>) from Dave Chinner in 2018 worked around the page-cache limitation by adding the `IOMAP_F_ZERO_AROUND` flag in [iomap](<https://docs.kernel.org/filesystems/iomap/design.html>). This flag pads I/O operations with zeroes if the size is less than a single block. The patches also removed the [`writepage()`](<https://elixir.bootlin.com/linux/v6.13.1/source/include/linux/fs.h#L400>) callback to ensure that the entire large block was written during the writeback. This effort was not upstreamed as [folios](<https://lwn.net/Articles/849538/>) were getting traction, which had the potential to solve the partial-eviction problem directly at the virtual filesystem layer. 

[Large folio support](<https://lwn.net/ml/linux-kernel/20220116121822.1727633-1-willy@infradead.org/>) was added to the page cache, and it was enabled in XFS in 5.18. If a filesystem supports large folios, then the page cache will opportunistically allocate larger folios in the read and write paths based on the size of the I/O operation. The filesystem calls [`mapping_set_large_folios()`](<https://elixir.bootlin.com/linux/v6.12.12/source/include/linux/pagemap.h#L435>) during inode initialization to enable the large-folios feature. But the page cache could still fall back to allocating an order-0 folio (a single page) if the system is running low on memory, so there is no guarantee on the size of the folios. 

The LBS patches, which were developed by me and some colleagues, add a way for the filesystem to inform the page cache of the minimum and maximum order of folio allocation to match its block size. The page cache will allocate large folios that match the order constraints set by the filesystem and ensure that no partial eviction of blocks occurs. The [`mapping_set_folio_min_order()`](<https://elixir.bootlin.com/linux/v6.12.12/source/include/linux/pagemap.h#L428>) and [`mapping_set_folio_order_range()`](<https://elixir.bootlin.com/linux/v6.12.12/source/include/linux/pagemap.h#L393>) APIs have been added to control the allocation order in the page cache. The order information is encoded in the `flags` member of the [`address_space` struct](<https://elixir.bootlin.com/linux/v6.12.12/source/include/linux/fs.h#L444>). 

Setting the minimum folio order is sufficient for filesystems to add LBS support since they only need a guarantee on the smallest folio allocated in the page cache. Filesystems can set the minimum folio order based on the block size during inode initialization. Existing callers of `mapping_set_large_folios()` will not notice any change in behavior because that function will now set the minimum order to zero and the maximum order to the `MAX_PAGECACHE_ORDER`. 

Under memory pressure, the kernel will try to break up a large folio into individual pages, which could violate the promise of minimum folio order in the page cache. The main constraint is that the page cache must always ensure that the folios in the page cache are never smaller than the minimum order. Since the 6.8 kernel, the memory-management subsystem has the support to split a large folio into any [lower-order folio](<https://lwn.net/ml/all/20240226205534.1603748-1-zi.yan@sent.com/>). LBS support uses this feature to always maintain the minimum folio order in the page cache even when a large folio is split due to memory pressure or truncation. 

#### Other filesystems?

Readers might be wondering if it will be trivial to add LBS support to other filesystems since the page cache infrastructure is now in place. The answer unfortunately is: it depends. Even though the necessary infrastructure is in place in the page cache to support LBS in any filesystem, the path toward adding this support depends on the filesystem implementation. XFS has been preparing for LBS support for a long time, which resulted in LBS patches requiring minimal changes in XFS. 

The filesystem needs to support large folios to support LBS. So any filesystem that is using buffer heads in the data path cannot support LBS at the moment. XFS developers moved away from buffer heads and designed iomap to address the shortcomings in buffer heads. While there is work underway to support large folios in buffer heads, it might take some time before it gets added. Once large folios support is added to the filesystem, adding LBS support is all about finding any corner cases that have made any assumption on block size in the filesystem. 

LBS has already found a use case in the kernel. The guarantee that the memory in the page cache representing a filesystem block will not be split has been used by the [atomic-write support in XFS](<https://lwn.net/ml/all/20241016100325.3534494-1-john.g.garry@oracle.com/>) for 6.13. The initial support in XFS will only allow writing one filesystem block atomically. The drive needs to be formatted with the desired size for atomic writes as its filesystem block size. 

The next focus in the LBS project is to [remove the logical-block-size restriction for block devices](<https://lwn.net/ml/all/20250204231209.429356-1-mcgrof@kernel.org/>). Similar to filesystem block size, the logical block size, which is the smallest size that a storage device can address, is restricted to the host page size due to limitations in the page cache. Block devices cache data in the page cache when applications performed buffered I/O operations directly on the devices and they use buffer heads by default to interact with the page cache. So large folio support is needed in buffer heads to remove this limitation for block devices. 

Many core changes, such as large folios support, XFS using iomap instead of buffer heads, multi-page bvec support in the block layer, and so on, took place over the past 17 years after the first LBS attempt. This resulted in adding LBS support with relatively fewer changes. With that being done for 6.12, XFS will finally support all the features that it supported in IRIX before it was ported to Linux in 2001\. 

[I would like to thank Luis Chamberlain and Daniel Gomez for their contributions to both this article and the LBS patches. Special thanks to Matthew Wilcox, Hannes Reinecke, Dave Chinner, Darrick Wong, and Zi Yan for their thorough reviews and valuable feedback that helped shape the LBS patch series.]
