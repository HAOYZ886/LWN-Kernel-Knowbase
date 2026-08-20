---
title: Optimizing Linux with cheap flash drives
url: https://lwn.net/Articles/428584/
date: "February 18, 2011"
category: "Solid-state storage devices"
author: "February 18, 2011 This article was contributed by Arnd Bergmann"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

February 18, 2011

This article was contributed by Arnd Bergmann

Flash drives are getting larger and cheaper; as a result, they are showing up in an increasing number of devices. These drives are not the same as the rotating-media drives which preceded them, and they have different performance characteristics. If Linux is to make proper use of of this class of hardware, it must drive it in a way which is aware of its advantages and disadvantages. 

This article will review the properties of typical flash devices and list some optimizations that should allow Linux to get the most out of low-cost flash drives. The kernel working group of the [Linaro project](<http://www.linaro.org>) is currently researching this topic as an increasing number of embedded designs move away from raw NAND flash devices to embedded MMC or SD drives that hide the NAND interface and provide a simplified linear block device. This drives down system design complexity and cost but also means that regular block-oriented filesystems are used instead of the Linux MTD layer that can talk to raw flash. 

Most filesystems and the block layer in Linux are highly optimized for rotating media, in particular by organizing all accesses to avoid seeks. It has become clear that some of these optimizations are pointless or even counterproductive with solid-state storage media. In recent kernels, there is a per-device flag for non-rotational devices that treats these slightly differently, by assuming that all seeks are free, but is that really enough to get good I/O performance on solid state drives? High-end drives are getting fast enough to make optimizations for CPU load more interesting than optimizations for ideal access patterns. In contrast, the more common SD cards and USB flash drives are very sensitive to specific access patterns and can show very high latencies for writes unless they are used with the preformatted FAT32 file layout. 

As an example, a desktop machine using a 16 GB, 25 MB/s CompactFlash card to hold an ext3 root filesystem ended up freezing the user interface for minutes during phases of intensive block I/O, despite having gigabytes of free RAM available. Similar problems often happen on small embedded and mobile machines that rely on SD cards for their file systems. 

To understand why this happens, it is important to find out how the embedded controllers on these cards work. Since very little information is publicly documented, most of the following information had to be gathered using reverse engineering based on timing data collected from a large number of SD cards and other devices. 

#### Pages, erase blocks and segments

All NAND flash chips are physically organized into "pages" and "erase blocks." A page is the smallest unit that can be addressed in a single read or write operation by the embedded microcontroller on a managed flash device, and it has an effective size between 2KB and 32KB in current consumer flash drives. This means that while a single 512-byte access is possible on the host interface (USB, ATA, MMC, ...), it takes almost the same time as a full page access inside of the drive. 

Although it is usually possible to write single pages, the data cannot be _over_ written without being erased first, and erasing is only possible in much larger units, typically between 128KB and 2MB. The controllers group these erase blocks into even larger segments, called "erase block groups," "allocation units," or simply "segments." The most common size for these segments is 4MB for drives in the multi-gigabyte class, and all operations on the drive happen in these units; in particular, the drive will never erase any unit smaller than a segment. 

The drives have a single lookup table which contains a mapping between logical segments and physical segments. On a typical 8GB SD card using 4MB segments, this table contains a little under 2000 entries, which is small enough to be kept in the RAM of the card's microcontroller at all times. A small number of physical segments is set aside in a pool to handle wear leveling, bad blocks and garbage collection. 

Ideally, the drive expects all data to be written in full segments, which is what happens when recording a live video or storing a music collection on a FAT32 filesystem. 

[![\[Bar chart\]](https://static.lwn.net/images/2011/arnd-ssd/extrememory-results-sm.png)](<https://lwn.net/Articles/428592/>) The way the physical characteristics of the card make themselves felt can be seen in the plot to the right (click on the thumbnail for the full-size version), which summarizes the results of a number of tests on an SDHC memory card. The best-case read throughput is 13.5MB/s, while the linear write throughput is 11.5MB/s. The results show that the segment size is 4MB; any properly-aligned, 4MB write will be fast. The smallest efficient block size for reads and writes is 64KB, all accesses smaller than that are significantly slower. Individual pages are 8KB; the costs of extra garbage collection caused by smaller writes can be seen. The card as a whole has been optimized for linear write operations; random writes are much slower. Additionally, only one segment can be open at a time; alternating between two segments will cause garbage collection at every access, slowing write speeds to a mere 33KB/s. That said, the FAT file table area (from 4MB to 8MB) is managed differently, enabling small writes to be done efficiently there. 

[![\[Performance plot\]](https://static.lwn.net/images/2011/arnd-ssd/panasonic-plot-sm.png)](<https://lwn.net/Articles/428836/>) The second image to the right shows a plot of read access times, in page granularity, on the first 32MB of a Panasonic Class 10 SDHC card. This plot illustrates various properties of the card. The segment size of 4MB can clearly be seen from the various changes in performance at the boundaries between segments. All closed segments have the same read performance, as do have all erased segments, which are a little faster to read. The FAT area in the second segment is a bit slower when reading because it uses a block remapping algorithm. One segment has been opened for writing by writing a few blocks in the middle before the read test, that segment can be seen as being a little faster to read on this specific card. Also, an effect of multi-level-cell (MLC) flash is that it alternates between slightly slower and faster pages, which the plot shows as two parallel lines for some segments. 

#### Wear leveling

When a segment that already contains data is written to, a new segment is allocated from the free pool and the drive writes the new data into that segment. Once the segment has been written to from start to finish, the lookup table will be updated to point to the new segment, while the old segment is put into the free pool and erased in the background. 

By always allocating a new segment, the drive can avoid wearing out a single physical segment in cases where the host always writes to the same block addresses. Instead, all writes are statistically distributed to all the segments that get written to from time to time. The better memory cards and SSDs also do static wear leveling, meaning they occasionally move a logical segment that contains static data to a physical segment that has been erased many times to even out the wear and increase the expected lifetime of the card. However, the vast majority of cheap memory cards do not do this but, instead, rely on the host software to write to every segment of the drive at some time or other. 

[![\[Segment mapping diagram\]](https://static.lwn.net/images/2011/arnd-ssd/segment-preview.png)](<https://lwn.net/Articles/428793/>) The diagram to the right shows how this mapping works in a typical flash drive; click on it for an animated version. 

To improve wear leveling, the host can also issue trim or erase commands on full segments to increase the size of the free pool. However, file systems in Linux do not know the segment size and typically issue trim commands on partial segments, which can improve write performance inside that segment but not help wear leveling across segments. 

#### Garbage Collection

In real life, writing 4 MB segments at once is more the exception than the rule, so drives need to cope with partial updates of segments. While data gets written to a logical segment, the controller normally has an old and a new physical segment associated with it. In order to free up the extra segment, it has to combine all the logical blocks in that segment into physical blocks on only one segment and discard all the previously used physical blocks, a process called garbage collection. A number of garbage collection techniques can be observed in current drives, including special optimizations using caching in RAM or NOR flash and dynamically adapting to the access patterns. 

Most drives however use a very simple garbage collection method, typically one of the following three. Each description below is accompanied by a diagram which, when clicked, will lead to an animated version showing how the technique works. 

**Linear-access optimized garbage collection**.  Drives that are advertised as being ideal for video storage usually expect long, contiguous reads and writes. They always write a physical segment from start to end, so, if the first write into a segment does not address the first logical block inside it, the drive copies all blocks in front of it from the old segment before writing the new data. Similarly, a subsequent write to a block that is not logically contiguous to the previously written one requires the drive to copy all intermediate blocks. 

[![\[Linear access diagram\]](https://static.lwn.net/images/2011/arnd-ssd/linear-preview.png)](<https://lwn.net/Articles/428796/>) Garbage collection simply fills the new segment up to the end with copies of the unchanged blocks from the old segment. 

The advantage is optimum performance for all reads and for long writes, but the disadvantage is that the drive ends up copying almost an entire segment for each block that gets written in the wrong order, for instance when the block elevator algorithm writes the blocks in reverse order attempting to avoid long seeks. Also, writing linear data smaller than the minimum block size of the drive makes it write the same block twice, which forces an immediate garbage collection. The minimum block size that the drive expects here is normally the cluster size of the preformatted FAT32 filesystem, between 4KB and 32KB, but on SD cards, it can be even larger than that. 

Drives that are hardwired to linear-access optimized segments are basically useless for ext3 and most other Linux filesystems because of this, because they keep small data structures like inodes and block bitmaps in front of the actual data and need to seek back to these in order to write new small files. 

**Block remapping**.  Fortunately, a significant number of flash drives support random access within a logical segment, by remapping logical blocks to free physical blocks [![\[Block remapping diagram\]](https://static.lwn.net/images/2011/arnd-ssd/random-preview.png)](<https://lwn.net/Articles/428831/>) as they get written. Since this requires maintaining another lookup mechanism, both read and write accesses are slightly slower than the ideal linear-access behavior, and a small amount of out-of-band data needs to be reserved to store the lookup table. 

This method also does not allow efficient writing in any small units when the manufacturers optimize for larger blocks in order to keep the size of the lookup table small. Writing the same block repeatedly still requires a full garbage-collection, which makes this method unsuitable for storing an ext3 journal or any other data that frequently gets written to the same area on the drive. 

**Data logging**.  The best random-access behavior is provided by using the same approach that log-structured filesystems like jffs2, logfs or nilfs2 and block-remappers like UBI in Linux use. Data that is written anywhere in the logical segment always goes to the next free block in the new physical segment, and the drive keeps a log of all the writes cached. Once the last free block is used up, a garbage collection is performed using a third physical segment. 

[![\[Data logging diagram\]](https://static.lwn.net/images/2011/arnd-ssd/logging-preview.png)](<https://lwn.net/Articles/428832/>) In the end, writing this way is slower than the other two approaches in the best case, because every block is written at least twice, but the worst case is much better. 

This approach is normally used only in the first few segments on the drive, which contain the file allocation table in FAT32 preformatted drives. Some drives are also able to use this mode when they detect access patterns that match writes to a FAT32 style directory entry. 

Obviously, any such optimizations don't normally do the right thing when a different filesystem is used on the drive than it was intended for, but there is some potential for optimization, e.g. by ensuring that the ext3 journal uses the blocks that are designed to hold the FAT. 

#### Restrictions on open segments

One major difference between the various manufacturers is how many segments they can write to at any given time. Starting to write a segment requires another physical segment, or two in case of a data logging algorithm, to be reserved, and requires some RAM on the embedded microcontroller to maintain the segment. Writing [![\[SSD thrashing\]](https://static.lwn.net/images/2011/arnd-ssd/thrashing-preview.png)](<https://lwn.net/Articles/428799/>) to a new segment will cause garbage collection on a previously open segment. That can lead to thrashing as the drive must repeatedly switch open segments; see the animation behind the diagram to the right for a visualization of how that works. 

On many of the better drives, five or more segments can be open simultaneously, which is good enough for most use cases, but some brands can only have one or two segments open at a time, which causes them to constantly go through garbage collection when used with most of the common filesystems other than FAT32. 

When a drive reserves the segments specifically to hold the FAT, these will always be open to allow updating it while writing streaming data to other segments. 

#### Partitioning

When a filesystem wants to optimize its block allocation to the geometry of a flash drive, it needs to know the position of the segments on the drive. On partitioned media, this also implies that each partition is aligned to the start of a segment, and this is true for all preformatted SD cards and other media that require special care for segment optimizations. 

Unfortunately, the fdisk and sfdisk tools from util-linux make it particularly hard to do this correctly, because they try to preserve an archaic geometry of 255 "heads" and 63 "sectors" and, by default, align partitions to "cylinder" boundaries. None of these units have any significance on today's hard drives or flash drives, but they are kept for backwards compatibility with existing software. The result is that most partitions are as misaligned as possible, they start on a odd-numbered 512-byte sector, which defeats all optimizations that a filesystem can do to align its accesses to logical blocks and segments inside of the partition. 

The same problem has been discussed a lot in the light of hard drives with 4KB sectors, but it is much more significant when dealing with flash media. Current versions of fdisk ask the kernel about physical sector (BLKPBSZGET) and optimum I/O size (BLKIOOPT), but currently these are rarely reported correctly by the kernel for flash drives, because the kernel itself does not have the necessary information. SDHC cards report the segment size in sysfs, but this is not used by any partitioning tools, and all cards currently seem to report 4MB segments, even those that actually use 2MB or 8MB segments internally. 

The linaro-media-create tool (from [Linaro Image Tools](<https://launchpad.net/linaro-image-tools>)) has recently been changed to align partitions to 4 MB boundaries when installing to a bootable SD card, to work around this problem. 

#### Future work

There is a huge potential for optimizing Linux to better deal with the deficiencies of flash media in various places in the kernel and elsewhere. With the storage and filesystem summit coming up this April, there is hopefully time to discuss these and other ideas: 

  * All partition tools should default to a much larger alignment, e.g. 4 MB or what the drive itself reports, for flash media and ignore cylinder boundaries. 

  * The page cache could benefit from the fact that larger accesses end up taking less time than accesses shorter than a flash page. When a drive reads 16KB, the kernel may as well add all of it to the page cache. 

  * The elevator and I/O scheduler algorithms can do much better than they do today for drives that only do linear access. Ideally, all outstanding writes to one segment should be submitted in order within a segment before moving to another segment. 

  * A stacked block device can be used to reorder blocks during write, creating a copy-on-write log-structured device on top of drives that can only write to one segment at a time. A first draft design for device is available on the [FlashDeviceMapper page](<https://wiki.linaro.org/WorkingGroups/Kernel/Projects/FlashDeviceMapper>) at Linaro. 

  * The largest potential is probably in the block allocation algorithm in the filesystem. The filesystem can ensure that it submits writes in the correct order to avoid garbage collection most of the time. Btrfs, nilfs2 and logfs get this right to a certain degree, but could probably get much better. 

#### Resources

More information about specific measurements can be found in the Linaro [flash card survey](<https://wiki.linaro.org/WorkingGroups/KernelConsolidation/Projects/FlashCardSurvey>). Readers are welcome to add data about their memory cards and USB drives to the list. 

The tool that was used to do all measurements is available from `git://git.linaro.org/people/arnd/flashbench.git`.
