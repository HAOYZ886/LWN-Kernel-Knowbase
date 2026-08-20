---
title: Moving the kernel to large block sizes
url: https://lwn.net/Articles/945646/
date: "September 27, 2023"
category: Block layer
author: "By Jake Edge September 27, 2023 OSSEU"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
September 27, 2023

* * *

[OSSEU](<https://lwn.net/Archives/ConferenceByYear/#2023-Open_Source_Summit_Europe>)

Using larger block sizes in the kernel for I/O is a recurring topic in storage and block-layer circles. The topic came up in [discussions](<https://lwn.net/Articles/933437/>) at the Linux Storage, Filesystem, Memory-Management and BPF Summit (LSFMM) back in May. One of the participants in those discussions, Hannes Reinecke, gave a talk at Open Source Summit Europe 2023 with an overview of the reasons behind using larger blocks for I/O, the current status of that work, and where it all might lead from here. 

Reinecke has worked at SUSE for "like an eternity, nearly 20 years now" and was involved with Linux before that; his first kernel was 1.1.5 or 1.0.5. More recently he has been involved with storage and with NVMe in particular. That led to a pet project of his "that has finally come to life", which is to be able to use larger blocks in Linux. 

#### Blocks and pages

Currently, Linux is restricted to using block sizes that are no larger than `PAGE_SIZE`, which is typically 4KB. But there are some systems and applications that would benefit from using larger pages; for example, some databases would really like to be able to work with chunks of 16KB because that is how they are organized internally. In addition, some hardware would benefit from handling data in larger block sizes because it reduces the amount of overhead needed to internally track the blocks, thus making the drives more efficient and cheaper. 

[ ![\[Hannes Reinecke\]](https://static.lwn.net/images/2023/osseu-reinecke-sm.png) ](<https://lwn.net/Articles/945729/>)

But, does there have to be a block size, he asked, couldn't the kernel just use whatever amount of data it wants to at the time? The problem is that there is no "do I/O" instruction that atomically reads or writes some arbitrary amount of data. Each I/O operation requires multiple instructions to set it up, transfer the data, and gather up the results. That increases the latency for each operation, so the goal is to minimize the number of I/O operations that are done, but there is a balance to be struck. 

There is a question of what the right size for these I/O operations should be. That was the subject of a lot of experimentation in the early days, he said. Eventually, researchers at University of California Berkeley ("of course, as usual") figured out that 512 bytes gave a reasonable compromise value between the overhead and I/O granularity. That was twenty years ago, at this point, but we still use 512 bytes—at least for now. 

CPUs have hardware-assisted memory management that operates in pages, however. There is support for determining which pages are dirty (i.e. need to be written to the backing store) that operate in page-size chunks, for example. That means the size of the page is CPU-dependent, Linux cannot just arbitrarily choose a size. For x86_64, the choices are 4KB, 2MB, or 1GB; for PowerPC and some Arm systems, 16KB is used as the page size. The kernel has a compile-time `PAGE_SIZE` setting that governs the size of the page. 

There is a need to read the pages in memory from disk, or to flush their contents to disk, at times. For buffered I/O, the page cache is what manages all of that; it uses the hardware-supplied dirty-page information to determine which pages need to be written. Since all of that is done at page granularity, it is natural to do I/O in page-size units. 

But if you had a number of consecutive pages that were all dirty, you could do I/O on the whole set of pages at once. Having a data structure that handles more than one page as a single unit would facilitate that, which is what [folios](<https://lwn.net/Articles/849538/>) are all about. Beyond buffered I/O, there is direct I/O, which user space has complete control over; the page cache is not involved and user space can do I/O in multiple blocks if it wants. Buffered I/O is provided by the filesystems via the page cache and there are a few different interfaces that can be used for that I/O. The oldest is buffer heads, its successor (of sorts) uses `struct bio`, and more recently there is iomap, which he said he would be getting back to. In order to do buffered I/O in larger sizes, though, the page cache needs to be converted to use folios. 

#### Folios

Folios are an effort to treat different kinds of pages in a common way. There are normal pages, compound pages (like an array of pages), and transparent hugepages (THPs), each of which has its own quirks. All of them can be addressed using a `struct page`, though, so kernel developers have to know whether a given page structure is actually a page—or something more complicated. A folio is explicitly designed to handle the different types and, importantly for his talk, it can represent more than a single page, thus allowing it to be used for larger block I/O. 

That requires converting the page cache—and probably the memory-management subsystem eventually—to use folios. That effort was proposed by Matthew Wilcox in 2020 and has been discussed at every LSFMM since. It has also been the subject of sometimes contentious mailing-list discussions over that span. But the work is ongoing and will be for several more years ("we will get there eventually"). He showed counts of "`struct page`" (8385) versus "`struct folio`" (1859) in the 6.4-rc2 kernel as a rough guide to where things stand. 

He then turned to buffer heads, which were present in the 0.01 kernel, so they are the original structure for I/O in Linux. Each buffer head is for a single 512-byte disk sector, it is linked to a particular page structure, and is internally cached in the buffer cache (to save on I/O when accessing it). Buffer heads are still in use by most filesystems and they are also used in a pseudo-filesystem for block devices. The page cache only came later in the kernel's history because the buffer cache was sufficient for the early days. 

A `struct buffer_head` is complicated, so in the 2.5 kernel, [`struct bio` was introduced](<https://lwn.net/Articles/26404/>) as a "basic I/O structure" for device drivers. It allows for vectorized I/O to or from an array of pages, routing and rerouting the bio structures to various devices, and is abstracted away from the page cache. These days, buffer heads are implemented on top of the bio infrastructure. There are a number of filesystems, such as AFS, CIFS, NFS, and FUSE, that use `struct bio` directly, thus do not rely on buffer heads. 

Finally, there is iomap "or Christoph Hellwig going crazy"; Hellwig got fed up with the existing I/O interfaces and created iomap as a replacement, Reinecke said. Iomap is a modern interface that already uses folios; it provides a way for a filesystem to specify how the I/O should be mapped and leaves the rest to the block layer. Several filesystems have been converted to use iomap, including XFS, Btrfs, and Zonefs, so nothing more needs to be done for those with regard to the folio conversion. One problem area for iomap, though, is documentation, which is somewhat hard to find and often out of date because iomap is under active development. 

#### Replacing buffer heads?

The storage community has long had a consensus that "buffer heads must die", he said. He led a [discussion on that topic](<https://lwn.net/Articles/931809/>) at this year's LSFMM. The thinking is that buffer heads are a legacy interface, using an ancient structure, so users should be converting to `struct bio` or iomap. But, a recent conversation on the ksummit-discuss mailing list contained a [disagreement from Linus Torvalds](<https://lwn.net/ml/ksummit-discuss/CAHk-=wg=xY6id92yS3=B59UfKmTmOgq+NNv+cqCMZ1Yr=FwR9A@mail.gmail.com/>). 

The vehemence of that response perhaps indicates that a different path should be chosen to get to the goal of larger block sizes, Reinecke said. Conversion to folios is useful, but only affects the page cache and the memory-management subsystem; buffer heads assume that I/O will be done in sub-page granularity (i.e. 512 bytes), so that needs to be addressed. One path might be to convert everything to iomap and then remove buffer heads, another would be to update buffer heads to work with larger I/O sizes. 

In an ideal world, all filesystems would be converted to use iomap, he said; it is a "modern interface and it is actually quite a nice interface". But, as the ksummit-discuss [thread](<https://lwn.net/ml/ksummit-discuss/ZO9NK0FchtYjOuIH@infradead.org/>) has shown, there are legacy filesystems that lack an active maintainer—or any maintainer at all. There is often little or no documentation for the legacy filesystems and no real way to test changes to them. Beyond that, converting any filesystems (legacy or not) is going to require better iomap documentation for the developers working on the conversions. 

Another possibility is to simply remove buffer heads; there is a patch set from Hellwig that [allows compiling buffer heads out of the kernel](<https://lwn.net/Articles/930173/>), which was merged for the 6.5 kernel. Turning that on would mean disabling all of the filesystems that use buffer heads, which is not entirely realistic at this point, Reinecke said. In particular, the FAT filesystem, which is needed for booting UEFI systems, would not be present in such a kernel. 

At LSFMM, Josef Bacik raised the idea of converting buffer heads to use folios, so that it could handle both sub-page and super-page I/O. While that is not the direction Reinecke would have chosen, he started to consider it. A conversion of that sort could either be fairly trivial, if the code was written without wholesale assumptions about sub-page I/O, or "it could be a complete nightmare" because that assumption is pervasive. 

Later that day, he was sitting at the bar after looking at the buffer heads code and "complaining bitterly" to his neighbor about them. He wondered how anyone could be expected to convert them, since they are so closely tied to pages. He then realized that his neighbor was Andrew Morton, who said: "back in the day when I wrote it, it was quite good—and it still works, doesn't it?" 

So, Reinecke started to reconsider the idea of converting buffer heads to folios, but there are a number of problems that need to be solved. For one thing, buffer heads and iomap are fundamentally incompatible. For example, there is a void pointer in the page structure that either points to a buffer head or an iomap structure, depending on which is being used; when looking at a page in the page cache, it is important to know which you have. The "mix and match approach" needs to be considered carefully. Reviewing the changes will be difficult, he said, because dependencies on `PAGE_SIZE` are hard to spot. 

All of that starts to make him wonder whether the overarching goal of I/O using larger block sizes is really worth all of this effort. "I think it is ... but that's just me." But he does know that databases really want to be able to do larger I/Os and the hope is that supporting larger I/Os will be more efficient for filesystems as well. For the most part, filesystems already do I/O in larger chunks. Beyond those benefits, the drive vendors would like to use larger blocks for efficiency, capacity, and, ultimately, cheaper devices. 

#### Progress

Reinecke had been working away on his patches and finished his [patch set](<https://lwn.net/ml/linux-fsdevel/20230918110510.66470-1-hare@suse.de/>) the previous week. As sometimes happens in the open-source world, though, another implementation surfaced around the same time. Luis Chamberlain and his colleagues at Samsung posted a [different patch set](<https://lwn.net/ml/linux-fsdevel/20230915213254.2724586-1-mcgrof@kernel.org/>) that covers much of the same ground. In the talk, Reinecke said that he was presenting his own patches to solve these problems, but that he would be working with the folks from Samsung on combining the two approaches in the near future. 

The overall idea is to switch buffer heads to be attached to a folio rather than to a page. That way, all of the I/O would still be smaller than the attached unit, so the assumptions in the buffer heads code would still be met. The folio would have a pointer to a single buffer head or to a list of buffer heads. There are some things that need to be kept in mind with this conversion; foremost is that the memory-management subsystem still works in units of `PAGE_SIZE`, while the page cache and buffer cache have moved to folios. 

But, in order to do I/O, buffer heads use the bio mechanism, which operates in 512-byte blocks. That is effectively hardwired throughout the block layer and its drivers—it is not something that can be changed without enormous effort, he said. But the actual I/O is handled by the lower-layer drivers, which already merge adjacent blocks into larger units. So the folios in the page cache can be handed to the block layer, which will enumerate them in 512-byte blocks, hand the results to the driver that will reassemble them into larger units. It all "should just work", even though it is not really the obvious way to attack the problem. 

So that is the core of what his patch set does. There was still other work to do, of course, including auditing the page cache to ensure that it is allocating folios of the size used by the underlying drive and to ensure that it is incrementing in folio-size steps, not by pages. He also needed to add an interface for the block drivers to report their block size to the page cache. It all worked well, perhaps even too well, since NFS wanted 128MB blocks—and got them—at least until the virtual machine hit an out-of-memory condition. That particular test "neatly proved that all large blocks leads to a higher memory fragmentation" if such a proof was actually needed. 

#### Done yet?

While it is great that these patches enable the kernel to talk to drives with large block sizes, there is a still a problem: there are no drives with large block sizes "because no one can talk to them". He has patches to update the block ramdisk driver (brd) to support larger blocks for testing purposes. That driver could then be used as the backing device for an NVMe target so that it could be tested with large block sizes. "That was quite cool but, of course, there is still some testing needed." 

There are still some pieces needed as well. QEMU needs to be updated to support large block sizes, the drivers need to be exercised using them, and other subsystems, such as SCSI, need to be tested. Beyond that, unification with the Samsung work will be required. Once that is all in hand, there will be reviews and the fallout from those to deal with as well before this work can go upstream. 

The memory-fragmentation issue is one that is still unresolved. Systems may well have devices with different block sizes in the future; 16KB should not be a major problem in this regard, but even larger block sizes are possible down the road. The memory-management layer continues to work in page-size chunks, which will lead to additional fragmentation. If systems could switch to using memory at the same granularity as the larger blocks, all would be well—but that assumes there is only one large block size, which may well not be true. 

One possible solution, which may be worth doing in its own right, is to switch the SLUB allocator to use higher-order folios, rather than page granularity. Then if `alloc_page()` users were converted to use SLUB, it would remove the fragmentation problem for allocations. Once again, though, that relies on there being a single large block size. He would be interested in hearing other ideas for improving the fragmentation situation in the presence of larger block sizes. 

He closed his talk with a suggestion "in case you are really bored": there is still the block layer and its 512-byte orientation that could be improved. Switching the block layer to use folios is not something for the faint of heart, but it should be doable, he thinks. The bio structure does not store the data directly, but uses a `struct bio_vec` for the data in a vectorized form. Those could perhaps be converted to use folios instead of pages, though there are some 4,000 uses of `bio_vec` in the block layer. 

[I would like to thank LWN's travel sponsor, the Linux Foundation, for travel assistance to Bilbao for OSSEU.]
