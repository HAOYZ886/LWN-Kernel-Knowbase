---
title: The return of RWF_UNCACHED
url: https://lwn.net/Articles/998783/
date: "December 4, 2024"
category: "Asynchronous IO; Memory management-Page cache"
author: "By Jonathan Corbet December 4, 2024"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
December 4, 2024

Linux offers two broad ways of performing I/O to files. Buffered I/O, which is the usual way of accessing a file, stores a copy of the transferred data in the kernel's page cache to speed future accesses. Direct I/O, instead, moves data directly between the storage device and a user-space buffer, avoiding the page cache. Both modes have their advantages and disadvantages. In 2019, Jens Axboe [proposed an uncached buffered mode](<https://lwn.net/Articles/806980/>) to get some of the advantages of both, but that effort stalled at the time. Now, [uncached buffered I/O is back](<https://lwn.net/ml/all/20241203153232.92224-2-axboe@kernel.dk>) with some impressive performance results behind it. 

By saving data in the page cache, buffered I/O can accelerate many I/O operations. Cached data need not be reread from the storage device, and multiple write operations can be combined in the cache, reducing the number of writes back to persistent storage. But that caching comes at a cost; the page cache is typically the largest user of memory on a Linux system, and the CPU must spend time copying data to and from the cache. Direct I/O avoids this memory use and copying, but it is inherently synchronous, adds complexity to a program, and provides a number of inconvenient pitfalls, especially in cases where a file is accessed concurrently. Developers normally only reach for direct I/O if they really need it. 

Still, as Axboe describes in the patch-set cover letter, users are often driven toward direct I/O, despite its challenges. That pressure is especially acute in cases where the data being transferred will not be needed again. Storing unneeded data in the page cache costs memory, but the problem is worse than that. Even though once-accessed data is put on the kernel's inactive list, meaning that it will be the first to be reclaimed when free memory runs low, the kernel must still make the effort to process that list and reclaim pages from it. With the right sort of I/O load (randomly accessing a set of files much larger than the system's RAM, for example), much of the available CPU time can be taken by the kernel's `kswapd` threads, which are working simply to reclaim memory from the page cache. 

The solution that he came up with in 2019 was to add a new flag, `RWF_UNCACHED`, for the [`preadv2()` and `pwritev2()`](<http://man7.org/linux/man-pages/man2/readv.2.html>) system calls. When that flag is present, those calls will perform I/O through the page cache as usual, with one exception: once the operation is complete, the relevant pages are immediately deleted from the page cache, making that memory available to the system without the need to go through the reclaim process. In 2019, the work then wandered into an attempt to avoid the page cache entirely, to get closer to direct-I/O performance, before coming to a stop. 

The new series picks things up again, returning to transferring data by way of the page cache and removing it afterward. For read operations, the data will be removed from the page cache as soon as it is copied into the user-space buffer (with the exception that, if it was already resident in the page cache prior to the operation, it will be left there afterward). Going through the page cache in this way avoids the coherency pitfalls that come with using direct I/O. 

Writes work like buffered writes do now; the data will be written to the page cache, and the pages will be marked for eventual writeback to persistent storage. Once that writeback completes, the pages will be removed from the page cache (except, again, in cases where they were resident there prior to the operation starting). In the meantime, though, multiple writes can be combined into a single writeback operation, maintaining I/O performance. 

Since the last time around, the kernel's file-I/O infrastructure has improved somewhat, to the point that much of the work of supporting `RWF_UNCACHED` can be performed in the kernel's [iomap](<https://docs.kernel.org/filesystems/iomap/index.html>) layer. Filesystems that use iomap fully will get `RWF_UNCACHED` support almost for free. Filesystems that use less generic code, including ext4, require a bit more work. The patch series includes the needed changes for ext4; XFS and Btrfs are supported as well. 

The effect of these changes can be seen in the associated benchmark results. For the read side, Axboe [included results](<https://lwn.net/ml/all/20241114152743.2381672-10-axboe@kernel.dk>) showing how, in the absence of `RWF_UNCACHED`, a system performing a lot of random reads will bog down once memory fills and reclaim begins. At that point, nearly 28 cores on the 32-core test system are busy just running `kswapd` full time. With `RWF_UNCACHED`, that bogging-down no longer happens, and `kswapd` does not appear among the top CPU-using processes as all. In summary: ""Not only is performance 65% better, it's also using half the CPU to do it"". The [write-side results](<https://lwn.net/ml/all/20241114152743.2381672-12-axboe@kernel.dk>) are almost the same. 

Most of the responses to this work have been positive; XFS developer Darrick Wong, for example, [said](<https://lwn.net/ml/all/20241112010157.GE9421@frogsfrogsfrogs/>) that ""there's plenty of places where this could be useful to me personally"". Dave Chinner (also an XFS developer) is less convinced, though. He [argued](<https://lwn.net/ml/all/ZzMLmYNQFzw9Xywv@dread.disaster.area/>) that, rather than adding a new flag for `preadv2()` and `pwritev2()`, Axboe should add the desired behavior to existing features in the kernel. Specifically, he said, the `POSIX_FADV_NOREUSE` flag to [`posix_fadvise()`](<https://man7.org/linux/man-pages/man2/posix_fadvise.2.html>) is meant to provide that functionality. Axboe, though, [disagreed](<https://lwn.net/ml/all/20b661ee-a7aa-4116-a0ec-96da9343af61@kernel.dk/>), saying that it is better to specify the desired behavior on each I/O operation than as an attribute of an open file: 

> Per-file settings is fine for sync IO, for anything async per-io is the way to go. It's why we have things like RWF_NOWAIT as well, where O_NONBLOCK exists too. I'd argue that RWF_NOWAIT should always have been a thing, and O_NONBLOCK is a mistake. That's why RWF_UNCACHED exists. 

In other words, the `O_NONBLOCK` flag to `open()` puts the election of non-blocking behavior in the wrong place. Rather than attaching that behavior to the file descriptor, it should be selected for specific operations. `RWF_UNCACHED` is a way to easily get that asynchronous behavior when needed. 

The discussion has since wound down for now, doubtless to be revived once the next version of the series is posted to the mailing lists. There would appear to be enough interest in this feature, though, to justify its merging into the mainline. It is too late to put uncached buffered I/O support into 6.13, but the chances of it showing up in 6.14 seem reasonably good.
