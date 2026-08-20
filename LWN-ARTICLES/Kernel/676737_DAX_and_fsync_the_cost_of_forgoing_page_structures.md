---
title: "DAX and fsync: the cost of forgoing page structures"
url: https://lwn.net/Articles/676737/
date: "February 24, 2016"
category: "DAX; Memory management-Nonvolatile memory"
author: "February 24, 2016 This article was contributed by Neil Brown"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

February 24, 2016

This article was contributed by Neil Brown

DAX, the support library that can help Linux filesystems provide direct access to persistent memory ([PMEM](<https://en.wikipedia.org/wiki/Persistent_memory>)), has seen substantial ongoing development since we [covered it](<https://lwn.net/Articles/610174/>) nearly 18 months ago. Its main goal is to bypass the page cache, allowing reads and writes to become memory copies directly to and from the PMEM, and to support mapping that PMEM directly into a process's address space with `mmap()`. Consequently, it was a little surprising to find that one of the challenges in recent months was the correct implementation of `fsync()` and related functions that are primarily responsible for synchronizing the page cache with permanent storage.

While that primary responsibility of `fsync()` is obviated by not caching any data in volatile memory, there is a secondary responsibility that is just as important: ensuring that all writes that have been sent to the device have landed safely and are not still in the pipeline. For devices attached using SATA or SCSI, this involves sending (and waiting for) a particular command; the Linux block layer provides the `blkdev_issue_flush()` API (among a few others) for achieving this. For PMEM we need something a little different.

There are actually two "flush" stages needed to ensure that CPU writes have made it to persistent storage. One stage is a very close parallel to the commands sent by `blkdev_issue_flush()`. There is a subtle distinction between PMEM "accepting" a write and "committing" a write. If power fails between these events, data could be lost. The necessary "flush" can be performed transparently by a memory controller using [Asynchronous DRAM Refresh (ADR) [PDF]](<http://www.snia.org/sites/default/files/NVDIMM%20Messaging%20and%20FAQ%20Jan%2020143.pdf#page=8>), or explicitly by the CPU using, for example, the new x86_64 instruction [`PCOMMIT`](<http://danluu.com/clwb-pcommit/>). This can be seen in the `wmb_pmem()` calls sprinkled throughout the DAX and PMEM code in Linux; handling this stage is no great burden. 

The burden is imposed by the other requirement: the need to flush CPU caches to ensure that the PMEM has "accepted" the writes. This can be avoided by performing "[non-temporal writes](<http://stackoverflow.com/questions/37070/what-is-the-meaning-of-non-temporal-memory-accesses-in-x86>)" to bypass the cache, but that cannot be ensured when the PMEM is mapped directly into applications. Currently, on x86_64 hardware, this requires explicitly flushing each cache line that might be dirty by invoking the `CLFLUSH` (Cache Line Flush) instruction or possibly a newer variant if available (`CLFLUSHOPT`, `CLWB`). An easy approach, referred to in discussions as the "[Big Hammer](<http://mid.gmane.org/1446070176-14568-1-git-send-email-ross.zwisler@linux.intel.com>)", is to implement the `blkdev_issue_flush()` API by calling `CLFLUSH` on every address of the entire persistent memory. While `CLFLUSH` is not a particularly expensive operation, performing it over potentially terabytes of memory was seen as worrisome.

The alternative is to keep track of which regions of memory might have been written recently and to only flush those. This can be expected to bring the amount of memory being flushed down from terabytes to gigabytes at the very most, and hence to reduce run time by several orders of magnitude. Keeping track of dirty memory is easy when the page cache is in use by using a flag in `struct page`. Since DAX bypasses the page cache, there are no page structures for most of PMEM, so an alternative is needed. Finding that alternative was the focus of most of the discussions and of the implementation of `fsync()` support for DAX, culminating in patch sets posted by Ross Zwisler ([original](<http://mid.gmane.org/1452103263-1592-1-git-send-email-ross.zwisler@linux.intel.com>) and [fix-ups](<http://mid.gmane.org/1453498573-6328-1-git-send-email-ross.zwisler@linux.intel.com>)) that [landed](<https://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=20c759ca98468d96d1fff8bd5e6753f458dbbfbd>) upstream for 4.5-rc1.

#### Is it worth the effort?

There was a subthread running through the discussion that wondered whether it might be best to [avoid the problem](<http://mid.gmane.org/CAPcyv4hjGYYPRyPjZc3CymmnSObB7mULRbeMZFjnKdoCD_m7pw@mail.gmail.com>) rather than fix it. A filesystem does not _have_ to use DAX simply because it is mounted from a PMEM device. It can selectively choose to use DAX or not based on usage patterns or policy settings (and, for example, would never use DAX on directories, as metadata generally needs to be staged out to storage in a controlled fashion). Normal page-cache access could be the default and write-out to PMEM would use non-temporal writes. DAX would only be enabled while a file is memory mapped with a new `MMAP_DAX` flag. In that case, the application would be explicitly requesting DAX access (probably using the [`nvml`](<https://github.com/pmem/nvml>) library) and it would take on the responsibility of calling `CLFLUSH` as appropriate. It is even conceivable that future processors could make cache flushing for a physical address range much more direct, so keeping track of addresses to flush would become pointless.

Dan Williams [championed](<http://mid.gmane.org/CAPcyv4hof4rVN0EZHhV9Q7VBE0WMw6hcSrLK-HvB5FOrOwY+tg@mail.gmail.com>) this position putting his case quite succinctly:

DAX in my opinion is not a transparent accelerator of all existing apps, it's a targeted mechanism for applications ready to take advantage of byte addressable persistent memory. 

He also expressed a concern that `fsync()` would end up being painful for large amounts of data.

Dave Chinner [didn't agree](<http://mid.gmane.org/20151103054039.GQ10656@dastard>). He provided a demonstration suggesting that the proposed overheads needed for `fsync()` would be negligible. He asserted instead: 

DAX is a method of allowing POSIX compliant applications get the best of both worlds - portability with existing storage and filesystems, yet with the speed and byte [addressablity] of persistent storage through the use of mmap. 

Williams' position resurfaced from time to time as it became clear that there were real and ongoing challenges in making `fsync()` work, but he didn't seem able to rally much support.

#### Shape of the solution

In general, the solution chosen is to still use the page cache data structures, but not to store `struct page` pointers in them. The page cache uses a [radix tree](<https://lwn.net/Articles/175432/>) that can store a pointer and a few tags (single bits of extra information) at every page-aligned offset in a file. The space reserved for the page pointer can be used for anything else by setting the least significant bit to mark it as an exception. For example, the tmpfs filesystem uses exception entries to keep track of file pages that have been written out to swap.

Keeping track of dirty regions of a file can be done by allocating entries in this radix tree, storing a blank exception entry in place of the page pointer, and setting the `PAGECACHE_TAG_DIRTY` tag. Finding all entries with a tag set is quite efficient, so flushing all the cache lines in each dirty page to perform `fsync()` should be quite straightforward.

As this solution was further explored, it was repeatedly found that some of those fields in `struct page` really are useful, so an alternative needed to be found.

#### Page size: `PG_head`

To flush "all the cache lines in each dirty page" you need to know how big the page is — it could be a regular page (4K on x86) or it could be a huge page (2M on x86). Huge pages are particularly important for PMEM, which is expected to sometimes be huge. If the filesystem creates files with the required alignment, DAX will automatically use huge pages to map them. There are even [patches](<http://mid.gmane.org/1454242175-16870-1-git-send-email-matthew.r.wilcox@intel.com>) from Matthew Wilcox that aim to support the direct mapping for extra-huge 1GB pages — referred to as "PUD pages" after the Page Upper Directory level in the [four-level page tables](<https://lwn.net/Articles/117749/>) from which they are indexed. 

With a `struct page` the `PG_head` flag can be used to determine the page size. Without that, something else is needed. Storing 512 entries in the radix tree for each huge page would be an option, but not an elegant option. Instead, one bit in the otherwise unused pointer field is used to flag a huge-page entry, which is also known as a "PMD" entry because it is linked from the Page Middle Directory.

#### Locking: `PG_locked`

The page lock is central to handling concurrency within filesystems and memory management. With no `struct page` there is no page lock. One place where this has [caused a problem](<http://mid.gmane.org/1449602325-20572-1-git-send-email-ross.zwisler@linux.intel.com>) is in managing races between one thread trying to sync a page and mark it as clean and another thread dirtying that page. Ideally, clean pages should be removed from the radix tree completely as they are not needed there, but attempts to do that have, so far, failed to avoid the race. Jan Kara [suggested](<http://mid.gmane.org/20160208134840.GC9451@quack.suse.cz>) that another bit in the pointer field could be used as a bit-spin-lock, effectively duplicating the functionality of `PG_locked`. That seems a likely approach but it has not yet been attempted.

#### Physical memory address

Once we have enough information in the radix tree to reliably track which pages are dirty and how big they are, we just need to know where each page is in PMEM so it can be flushed. This information is generally of little interest to common code so handling it is left up to the filesystem. Filesystems will normally attach something to the `struct page` using the `private` pointer. In filesystems that use the `buffer_head` library, the `private` pointer links to a `buffer_head` that contains a `b_blocknr` field identifying the location of the stored data.

Without a `struct page`, the address needs to be found some other way. There are a number of options, several of which have been explored. The filesystem could be asked to perform the lookup from file offset to physical address using its internal indexing tables. This is an indirect approach and may require the filesystem to reload some indexing data from the PMEM (it wouldn't use direct-access for that). While the first patch set used this approach, it did not survive long.

Alternately, the physical address could be stored in the radix tree when the page is marked as dirty; the physical address will already be available at that time as it is just about to be accessed for write. This leads to another question: exactly how is the physical address represented? We could use the address where the PMEM is mapped into the kernel address space, but that leads to [awkward races](<http://mid.gmane.org/CAPcyv4irspQEPVdYfLK+QfW4t-1_y1gFFVuBm00=i03PFQwEYw@mail.gmail.com>) when a PMEM device is disabled and unmapped. Instead, we could use a sector offset into the block device that represents the PMEM. That is what the current implementation does, but it implicitly assumes there is just one block device, or at least just one per file. For a filesystem that integrates volume management (as Btrfs does), this may not be the case.

Finally, we could use the page frame number (PFN), which is a stable index that is assigned by the BIOS when the memory is discovered. Wilcox [has patches](<http://mid.gmane.org/20160131023247.GZ2948@linux.intel.com>) to move in this direction, but the work is ~~70%~~ maybe [50%](<http://mid.gmane.org/20160201134410.GD2948@linux.intel.com>) done. Assuming that the PFN can be [reliably mapped](<http://mid.gmane.org/20160131180738.GB2948@linux.intel.com>) to the kernel address that is needed for `CLFLUSH`, this seems like the best solution.

#### Is this miniature `struct page` enough?

One way to look at this development is that a 64-bit miniature `struct page` has been created for the DAX use case to avoid the cost of a full `struct page`. It currently contains a "huge page" flag and a physical sector number. It may yet gain a lock bit and have a PFN in place of the sector number. It seems prudent to ask if there is anything else that might be needed before DAX functionality is complete.

As quoted above, Chinner appears to think that transparent support for full POSIX semantics should be the goal. He went on to [opine](<http://mid.gmane.org/20160203075411.GC459@dastard>) that:

This is just another example of how yet another new-fangled storage technology maps precisely to a well known, long serving storage architecture that we already have many, many experts out there that know to build reliable, performant storage from... :) 

Taking that position to its logical extreme would suggest that anything that can be done in the existing storage architecture should work with PMEM and DAX. One such item of functionality that springs to mind is the [`pvmove`](<http://man7.org/linux/man-pages/man8/pvmove.8.html>) tool. When a filesystem is built on an LVM2 volume, it is possible to use `pvmove` to move some of the data from one device to another, to balance the load, decommission old hardware, or start using new hardware. Similar functionality could well be useful with PMEM.

There would be a number of challenges to making this work with DAX, but possibly the biggest would be tearing down memory mappings of a section of the old memory before moving data across to the new. The Linux kernel has some infrastructure for memory [migration](<https://lwn.net/Articles/157066/>) that would be a perfect fit — if only the PMEM had a table of `struct page` as regular memory does. Without those page structures, moving memory that is currently mapped becomes a much more interesting task, though likely not an insurmountable one.

On the whole, it seems like DAX is showing a lot of promise but is still in its infancy. Currently, it can only be used on ext2, ext4, and XFS, and only where they are directly mounted on a PMEM device (i.e. there is no LVM support). Given the recent rate of change, it is unlikely to stay this way. Bugs will be fixed, performance will be improved, coverage and features will likely be added. When inexpensive persistent memory starts appearing on our motherboards it seems that Linux will be ready to make good use of it.
