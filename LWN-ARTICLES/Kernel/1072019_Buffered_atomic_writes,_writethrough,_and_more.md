---
title: "Buffered atomic writes, writethrough, and more"
url: https://lwn.net/Articles/1072019/
date: "May 14, 2026"
category: Atomic IO operations
author: "By Jake Edge May 14, 2026 LSFMM+BPF"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 14, 2026

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2026/>)

In back-to-back sessions at the start of the 2026 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) (which spilled over into a third slot), the [atomic-buffered-writes feature](<https://lwn.net/Articles/1060063/>) was discussed. In the first session, Pankaj Raghav and Andres Freund set the stage with an introduction to the problem, along with a use case for its solution: the PostgreSQL database system. In the second, Ojaswin Mujoo described a potential way forward for the feature using an approach based on writethrough, which effectively means that the kernel immediately writes the data to disk instead of waiting for writeback from the page cache to occur. As might be expected, there was quite a bit of discussion among the assembled filesystems and storage developers during the combined sessions for those tracks. 

[ ![\[Andres Freund\]](https://static.lwn.net/images/2026/lsfmb-freund-sm.png) ](<https://lwn.net/Articles/1072681/>)

Freund began by describing what PostgreSQL currently does [to prevent torn (partial) writes](<https://transactional.blog/blog/2025-torn-writes>) of its 8KB default pages; without atomic guarantees from the block layer, it uses its [full-page-writes](<https://wiki.postgresql.org/wiki/Full_page_writes>) feature to write to the write-ahead log (WAL) in order to ensure that the pages are fully written. There are a number of benefits that would come with atomic 8KB writes: for example, turning off full page writes results in around 1.7x more transactions per second (TPS) with a 14x reduction in the variability of the TPS as well. It would also reduce the write amplification factor (WAF) because the pages would not need to be written twice to ensure their integrity on disk. Meanwhile, [continuous archiving with point-in-time recovery](<https://www.postgresql.org/docs/current/continuous-archiving.html>) requires storing the WAL in the archive logs, which can add up to a significant amount of storage quickly, he said. A simple benchmark showed a 14x increase in accumulated WAL size when full page writes is used. 

An attendee asked what atomic page size PostgreSQL needed. Freund said that PostgreSQL would like to be able to write in larger chunks than 8KB, but that it only needed assurance that 8KB chunks would not be torn. Randy Jennings asked if the storage needed to be able to support these atomic writes; Freund said that it was required for the atomic-write feature. 

Ted Ts'o pointed out that the ""whole atomic-writes saga"" has been going on for years; it has featured in multiple prior summits, for example. It started, he said, because the software-defined storage used by the cloud vendors could provide the atomic guarantees needed by databases that were using direct I/O; performance improvements of 2x were common using that feature. Now, databases that use buffered writes, like PostgreSQL, would like to get those same performance boosts, which is why the buffered-atomic-writes feature has arisen. That is especially true now that NVMe devices can provide the same atomic-write granularity as has been available for cloud storage, he said. 

[ ![\[Pankaj Raghav\]](https://static.lwn.net/images/2026/lsfmb-raghav-sm.png) ](<https://lwn.net/Articles/1072682/>)

Raghav explained why writeback, which is used for buffered I/O, is fundamentally incompatible with atomic writes. Leaving pages in the page cache until writeback occurs provides a large window in time where the pages could be modified. Also, if there is memory pressure, page faults or reclaim could potentially result in a torn write. 

So a new flag, `RWF_WRITETHROUGH`, has been proposed for [`pwritev2()`](<https://man7.org/linux/man-pages/man2/pwritev2.2.html>), which will copy the data to the page cache and immediately issue a direct I/O to write the data from the provided buffer to the block device. Matthew Wilcox asked how the new flag differs from `RWF_DSYNC`; Jan Kara said that `RWF_DSYNC` provides a much stronger guarantee that the write operation has completed successfully all the way down to the disk. `RWF_WRITETHROUGH` simply indicates that the write has been issued. The `RWF_WRITETHROUGH` flag does not implement the atomic behavior; that would have to be added as a separate feature using the flag, Raghav said. 

Christoph Hellwig said that he is not a fan of PosgreSQL's use of buffered I/O, which drives the need for buffered atomic writes, but did not have a problem with `RWF_WRITETHROUGH`. ""I think he said 'no objection'"", Amir Goldstein said to laughter. 

John Garry wondered how the `pwritev2()` call could know for sure that the hardware can support an atomic write and determine what alignment is required to perform it. Raghav said that the plan was to use the same mechanism that is used for direct I/O (`O_DIRECT`) atomic writes. Ts'o said that the problem had already been solved for `O_DIRECT`; the alignment requirements come from the block device and using `pwritev2()` with the `RWF_ATOMIC` flag says that all of the buffers are properly aligned. 

Wilcox suggested starting out small, perhaps by implementing `RWF_DSYNC` using the direct I/O that `RWF_WRITETHROUGH` does after copying the data to the page cache. But Hellwig did not think that literally using the direct-I/O path from the write made sense; simply implementing it with a regular block write might be better. The direct-I/O path handles a lot of corner cases that may not need to be considered, but he said that it was only a ""gut feeling"" so it was worth trying it to see. He noted that some developers have complained about the complexity of the direct-I/O code path, so maybe this effort could be a starting point for a simpler interface. 

#### Writethrough

Next up was Mujoo, who began with a timeline of support for atomic writes in the kernel. For direct I/O, Linux 6.13 in January 2025 could write a single filesystem block atomically. In June 2025, the 6.16 kernel added the ability to handle multiple filesystem blocks atomically. On the buffered-I/O side, there have been three designs, culminating in the writethrough approach in April 2026. The first two designs suffered from the problem that there is no easy way for a given write operation to communicate with the writeback path to ensure that it is treated atomically. 

[ ![\[Ojaswin Mujoo\]](https://static.lwn.net/images/2026/lsfmb-mujoo-sm.png) ](<https://lwn.net/Articles/1072683/>)

As noted in the previous session, the writethrough approach avoids that problem by immediately initiating the I/O to the device from the `pwritev2()` call. That avoids the need to track atomic ranges for page-cache pages. Tracking those ranges made the first two designs more complex, so combining the write path with the I/O-submission path looks to simplify things. There may be other uses beyond atomic writes that can use the same technique, he said. 

He stepped through a flow chart that described the writethrough mechanism. From the write, the data is copied into the page cache, a [`bio_vec`](<https://elixir.bootlin.com/linux/v7.0.6/source/include/linux/bvec.h#L19>) is created from the folio range, and an I/O operation is initiated. If it is not an asynchronous write, the initial write awaits the completion of the block I/O and then returns to the caller. For an asynchronous I/O, the I/O completion is handled by a workqueue in the background, similar to the way it is done for direct I/O. 

There are several use cases, he said. Buffered atomic writes can be based on writethrough without needing extra code to track atomic ranges. Writes with `RWF_DSYNC` can use writethrough to support asynchronous buffered writes by moving the call to [`generic_write_sync()`](<https://elixir.bootlin.com/linux/v7.0.6/source/include/linux/fs.h#L2648>) from the submission to the completion phase; writethrough provides the shared context between the two phases. Likewise, other use cases that need tracking between the write and the writeback, such as `RWF_DSYNC` with forced unit access (FUA) and `RWF_DONTCACHE` writes, may be able to use writethrough. 

He put up some performance graphs that showed 35-60% improvements in write speed when doing random writes to separate files from multiple threads. But they also showed up to a 65% performance decrease when all of the threads are writing to the same file. He believes that is due to contention on the inode lock, which is held within the I/O-submission path. 

Ts'o wondered if that performance degradation would be a problem in practice for PostgreSQL. Freund said that there are multiple threads writing, but it is not clear to him whether there would be a lot of concurrent writing to the same file with real workloads. 

Hellwig thought that the critical section holding the inode lock could be reduced, as there is no real reason to hold it throughout the I/O-submission process. Mujoo said that he could look at splitting out some of the code that currently runs with the inode lock held, which Hellwig thought should be doable and might help simplify some other code paths. 

There was some discussion of the need to prevent writeback occurring on the pages in the page cache while the writethrough operation is being performed. There is a need to prevent modification of the in-flight buffer, which could interfere with checksums and the like, Hellwig said; that can be accomplished by taking the writeback lock. The default mode for the [iomap layer](<https://docs.kernel.org/filesystems/iomap/>) is to submit its I/O without holding the page lock, just the writeback lock, he said. 

The discussion turned to reducing contention on the [`xa_lock`](<https://www.kernel.org/doc/html/latest/core-api/xarray.html#locking>) and possibly using a shared lock for aligned, buffered I/O, as is used for direct I/O. Kara seemed to think it was a reasonable idea and Hellwig suggested that the right time to introduce such a lock was when a new flag (e.g. `RWF_WRITETHROUGH`) was being added. The proposed writethrough feature is much the same as `RWF_DSYNC`, Hellwig said, except that it does not guarantee that the block device actually flushes its cache to disk. If a shared lock is added later, a new flag will need to be added at that time to govern its use, so it makes sense to just include that with the writethrough feature in his mind. 

#### io_uring

Josef Bacik said that he hated having this kind of conversation and wanted to step back and look toward avoiding all of the various special cases in the I/O path that are handled by adding new flags. ""What we should be doing here is just rethinking how we do this""; he suggested looking at [io_uring](<https://man7.org/linux/man-pages/man7/io_uring.7.html>) as a potential solution. Each of the low-level operations needed for I/O (update the page cache, submit the I/O operation, etc.) would be a separate io_uring operation that can be built up into whatever style of I/O user space wants. 

Doing so would allow user space to ""mix and match all of these different features"" and avoid needing the kernel developers to figure out how new `pwritev2()` flags interact for all of the different special cases. Christian Brauner was concerned that the same problems would eventually arise for an io_uring-based solution; in a few years, there would need to be discussions about how the different operations interact. Hellwig did not see why moving the problem into io_uring would make the situation any better, though he found Bacik's description to be fairly abstract. 

Goldstein worried that it would be difficult for users to understand how to use the io_uring interface; meanwhile, the flags are a way to limit the number of different ways the operations can be combined, so moving to io_uring would substantially increase the testing matrix required. Bacik said that every new use case brings a slight wrinkle to how the write operation interacts with the page cache, writeback, iomap, and so on; of the full matrix that might be exposed, the flags limit the choices, but that ends up leading to more flags. 

I/O testing already ""sucks"" and moving to io_uring would potentially make that worse, he said. But ""it pushes the complexity of what user space wants down to user space"", which is better than cementing various heuristics and policy decisions into the kernel code. That results in user-space developers being unhappy because their specific use case is not one of the ones supported—followed by another flag proposal and long discussions on the mailing list and at the summit. 

Ts'o said that they would need to see the code before being able to determine whether the approach was viable, but his concern was that locks would need to be taken multiple times for the various sub-operations. There might be a way to analyze the series of operations in order to optimize the locking, but without that, performance may suffer compared to what there is today. Analyzing the operations and combining the lock acquisition would also increase the test matrix because it would make things dependent on the order of operations ""and this scares me"". 

Hellwig wondered what concrete io_uring operations were being talked about, as he still found the discussion to be too abstract. Brauner asked: ""Why is Jens [Axboe] so silent?"". That was met with laughter and, eventually, a response from Axboe, who is the io_uring maintainer, but it was clear that he did not have strong feelings about the idea. Bacik said that it simply did not make sense to ""add a new flag every two years"" to do some ""new, special thing"". 

The complexity of the new io_uring commands led to worries that most developers would not be able to use them; there is a reason that the kernel developers are defining how the different kinds of I/O should be done. Bacik said that the synchronous-I/O options could continue being handled with flags of various sorts, but that newer, fancier asynchronous use cases could be pointed at the io_uring-based approach. 

Hellwig was somewhat skeptical that the various operations could be fully defined with their semantics clearly specified, but even then the testing becomes burdensome. Others were not so sure that it would change the testing picture all that much in comparison to all of the existing flags and combinations of them. Brauner worried that changes to the semantics of writes would continue and that in five years, say, the same kinds of discussions would have to happen for an io_uring-based solution; it may just be kicking the can down the road. 

After Axboe pointed out that system calls could be added instead, Bacik wondered, perhaps not entirely seriously, if instead of io_uring he should have proposed ""17 new syscalls""; he simply thought that io_uring ""sounded better"". He is concerned that trying to shoehorn all of a feature under a single all-encompassing flag leads to problems; finding a way to split up the pieces that are composed to perform the I/O would make more sense. 

There was some discussion of the overall design of the I/O API that currently exists versus what might be done differently if the kernel developers were to start over. For example, Ts'o noted that `O_DIRECT` was designed by Oracle several decades ago and filesystems implement it differently because it was not clearly specified. But any overhaul of the API will not be used widely for multiple years and, meanwhile, the current API will have to be maintained. 

Goldstein summarized that part of the discussion as the session concluded by saying that adding a new flag is probably the easier approach because user-space developers are used to the system-call API and understand it. But the flag should be well-documented first, so that reviewers can try to ensure that it makes sense and fits with everything else. If it cannot be clearly specified, that is a pretty clear indication that the feature is not on the right track. 

[I would like to apologize for any errors here. The acoustics in the room were problematic for both hearing and recording. Misunderstanding and misidentification may have resulted.]
