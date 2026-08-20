---
title: "JFFS2, UBIFS, and the growth of flash storage"
url: https://lwn.net/Articles/528617/
date: "December 11, 2012"
category: "Filesystems-Flash"
author: "December 11, 2012 This article was contributed by Neil Brown"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

December 11, 2012

This article was contributed by Neil Brown

When thinking about filesystems for modern flash storage devices, as we have recently done with [f2fs](<https://lwn.net/Articles/518988/>) and [NILFS2](<https://lwn.net/Articles/522507/>), two other filesystems that are likely to quickly spring to mind, and be almost as quickly discarded, are JFFS2 and UBIFS. They spring to mind because they were designed specifically to work with flash, and are discarded because they require access to "raw flash" whereas the flash devices we have been considering have a "flash translation layer" (FTL) which hides some of the details of the flash device and which needs to be accessed much like a disk drive. 

This quick discarding may well not be appropriate — these are open-source filesystems after all and are thus free to be tinkered with. If the Apollo 13 technicians were [able to](<http://en.wikipedia.org/wiki/Apollo_13#Crew_survival_and_return_journey>) link the Lithium hydroxide canisters from the command module to the CO₂ scrubber in the Lunar module, it shouldn't be too hard for us to link a raw-flash filesystem to a FTL based storage chip — if it seemed like a useful thing to do. 

Raw access to a flash device goes through the "[mtd](<http://www.linux-mtd.infradead.org/>)" (Memory Technology Devices) interface in Linux and, while this is a rich interface, the vast majority of accesses from a filesystem are via three functions: `mtd_read()`, `mtd_write()` and `mtd_erase()`. The first two are easily implemented by a block device — though you need to allow for the fact that the mtd interface is synchronous while the block layer interface is asynchronous — and the last can be largely ignored as an FTL handles erasure internally. In fact, Linux provides a "block2mtd" device which will present an arbitrary block device as an mtd device. Using this might not be the most efficient way to run a filesystem on new hardware, but it would at least work as a proof-of-concept. 

So it seems that there could be some possibility of using one of these filesystems, possibly with a little modification, on an FTL-based flash device, and there could certainly be value in understanding them a little better as, at the very least, they could have lessons to teach us. 

#### A common baseline

Despite their separate code bases, there is a lot of similarity between JFFS2 and UBIFS — enough that it seems likely that the latter was developed in part to overcome the shortcomings of the former. One similarity is that, unlike the other filesystems we have looked at, neither of these filesystems has a strong concept of a "basic block size". The concept is there if you look for it, but it isn't prominent. 

One of the main uses of a block size in a filesystem is to manage free space. Some blocks are in use, others are free. If a block is only partially used — for example if it contains the last little bit of a file — then the whole block is considered to be in use. For flash filesystems, blocks are not as useful for free-space management as this space is managed in terms of "erase blocks," which are much larger than the basic blocks of other filesystems, possibly as large as a few megabytes. Another use of blocks in a filesystem is as a unit of metadata management. For example NILFS2 manages the `ifile` (inode file) as a sequence of blocks (rather than just a sequence of inodes), while F2FS manages each directory as a set of hash tables, each of which contains a fixed number of blocks. 

JFFS2 and UBIFS don't take this approach at all. All data is written consecutively to one or more erase blocks with some padding to align things to four-byte boundaries, but with no alignment so large that it could be called a block. When indexing of data is needed, an erase-block number combined with a byte offset meets the need, so the lack of alignment does not cause an issue there. 

Both filesystems further make use of this freedom in space allocation by compressing the data before it is written. Various compression schemes are available including LZO and ZLIB together with some simpler schemes like run-length encoding. Which scheme is chosen depends on the desired trade off between space saving and execution time. This compression can make a small flash device hold nearly twice as much as you might expect, depending on the compressibility of the files of course. Your author still recalls the pleasant surprise he got when he found out how much data would fit on the JFFS2 formatted 256MB flash in the original [Openmoko Freerunner](<http://wiki.openmoko.org/wiki/Neo_FreeRunner_Hardware#Flash>): a reasonably complete Debian root filesystem with assorted development tools and basic applications still left room for a modest amount of music and some [OSM](<http://www.openstreetmap.org/>) map tiles. 

In each case, the data and metadata of the filesystem are collected into "nodes" which are concatenated and written out to a fresh erase block. Each node records the type of data (inode, file, directory name, etc), the address of the data (such as inode number), the type of compression and a few other details. This makes it possible to identify the contents of the flash when mounting and when cleaning, and effectively replaces the "segment summary" that is found in f2fs and NILFS2. 

Special note should be made of the directory name nodes. While the other filesystems we have studied store a directory much like a file, with filenames stored at various locations in that file, these two filesystems do not. Each entry in the directory is stored in its own node, and these nodes do not correspond to any particular location in a "file" — they are simply unique entries. JFFS2 and UBIFS each have their own particular way of finding these names as we shall see, but in neither case is the concept of a file offset part of that. 

The one place where a block size is still visible in these filesystems is in the way they chop a file up into nodes for storage. In JFFS2, a node can be of any size up to 4KB so a log file could, for example, be split up as one node per line. However the current implementation always writes whole pages — to quote the in-line commentary, ["It sucks, but it's simple"](<http://lxr.linux.no/linux+v3.6.6/fs/jffs2/file.c#L238>). For UBIFS, data nodes must start at a 4KB-aligned offset in the file so they are typically 4KB in size (before compression) except when at the end of the file. 

#### JFFS2 — the journaling flash filesystem

A traditional journaling filesystem, such as ext3 or xfs, adds a journal to a regular filesystem. Updates are written first to the journal and then to the main filesystem. When mounting the filesystem after a shutdown, the journal is scanned and anything that is found is merged into the main filesystem, thus providing crash tolerance. JFFS2 takes a similar approach with one important difference — there is no "regular filesystem". With JFFS2 there is only a journal, a journal that potentially covers the entire device. 

It is probably a little misleading to describe JFFS2 as "just one journal". This is because it might lead you to think that when it gets to the end of the journal it just starts again at the beginning. While this was true of JFFS1, it is not for JFFS2. Rather it might be clearer to think of each erase block as a little journal. When one erase block is full, JFFS2 looks around for another one to use. Meanwhile if it notices that some erase blocks are nearly empty it will move all the active nodes out of them into a clean erase block, and then erase and re-use those newly-cleaned erase blocks. 

When a JFFS2 filesystem is mounted, all of these journals, and thus the entire device, are scanned and every node found is incorporated into an in-memory data structure describing the filesystem. Some nodes might invalidate other nodes; this may happen when a file is created and then removed: there will be a node recording the new filename as belonging to some directory, and then another node recording that the filename has been deleted. JFFS2 resolves all these modifications and ends up with a data structure that describes the filesystem as it was that last time something was written to it, and also describes where the free space is. The structure is kept as compact as possible and naturally does not contain any file data; instead, it holds only the addresses where the data should be found and so, while it will be much smaller than the whole filesystem, it will still grow linearly as the filesystems grows. 

This need to scan the entire device at mount time and store the skeleton of the filesystem in memory puts a limit on the size of filesystem that JFFS2 is usable for. Some tens of megabytes, or even a few hundred megabytes, is quite practical. Once the device gets close to, or exceeds, a gigabyte, JFFS2 become quite impractical. Even if memory for storing the tree were cheap, time to mount the filesystem is not. 

This is where UBIFS comes in. While the details are quite different, UBIFS is a lot like JFFS2 with two additions: a tree to index all the nodes in the filesystem, and another tree to keep track of free space. With these two trees, UBIFS avoids both the need to scan the entire device at mount time and the need to keep a skeleton of the filesystem in memory at all times. This allows UBIFS to scale to much larger filesystems — certainly many tens of gigabytes and probably more. 

But before we look too closely at these trees it will serve us well to look at some of the other details and in particular at "UBI", a layer between the MTD flash interface layer and UBIFS. UBI uses an unsorted collection of flash erase blocks to present a number of file system images; UBI stands for Unsorted Block Images. 

#### UBI — almost a Flash Translation Layer

The [documentation](<http://www.linux-mtd.infradead.org/doc/ubi.html>) for UBI explicit states that it is **not** a flash translation layer. Nonetheless it shares a lot of functionality with an FTL, particularly wear leveling and error management. If you imagined UBI as an FTL where the block size was the same as the size of an erase block, you wouldn't go far wrong. 

UBI uses a flash device which contains a large number of Physical Erase Blocks (PEBs) to provide one or more virtual devices (or "volumes") which each consist of a smaller number of Logical Erase Blocks (LEBs), each slightly smaller than a PEB. It maintains a mapping from LEB to PEB and this mapping may change from time to time due to various causes including: 

  * **Writing to an LEB.** When an LEB is written, the data will be written to a new, empty, PEB and the mapping from LEB to PEB will be updated. UBI is then free to erase the old PEB at its leisure. Normally, the first new write to an LEB will make all the data previously there inaccessible. However, a feature is available where the new PEB isn't committed until the write request completes. This ensures that after a sudden power outage, the LEB will either have the old data or the complete new data, never anything else. 

  * **Wear leveling.** UBI keeps a header at the start of each PEB which is rewritten immediately after the block is erased. One detail in the header is how many times the PEB has been written and erased. When UBI notices that the difference between the highest write count and the lowest write count in all the PEBs gets too high (based on a compile-time configuration parameter: **[MTD_UBI_WL_THRESHOLD](<http://lxr.linux.no/#linux+v3.6.6/drivers/mtd/ubi/Kconfig#L13>)**), it will move an LEB stored in a PEB with a low write count (which is assumed to be stable since the PEB containing it has not been rewritten often) to one with a high write count. If this data continues to be as stable as it has been, this will tend to reduce the variation among write counts and achieve wear leveling. 

  * **Scrubbing.** NAND flash includes an error-correcting code (ECC) for each page (or sub-page) which can detect multiple-bit errors and correct single-bit errors. When an error is reported while reading from a PEB, UBI will relocate the LEB in that PEB to another PEB so as to guard against a second bit error, which would be uncorrectable. This process happens transparently and is referred to as "scrubbing". 

The functionality described above is already an advance on the flash support that JFFS2 provides. JFFS2 does some wear leveling but it is not precise. It keeps no record of write counts but, instead, decides to relocate an erase-block based on the roll of a dice (or actually the sampling of a random number) instead. This probably provides some leveling of wear, but there are no guarantees. JFFS2 also has no provision for scrubbing. 

The mapping from PEB to LEB is stored spread out over all active erase blocks in the flash device. After the PEB header that records the write count there is a second header which records the volume identifier and LEB number of the data stored here. To recover this mapping at mount time, UBI needs to read the first page or two from every PEB. While this isn't as slow as reading every byte like JFFS2 has to, it would still cause mount time to scale linearly with device size — or nearly linearly as larger devices are likely to have larger erase block sizes. 

Recently this situation has improved. A new [feature](<https://lwn.net/Articles/517422/>) known has "fastmap" made its way into the UBI driver for Linux 3.7. Fastmap stores a recent copy of the mapping in some erase block together with a list of the several (up to 256) erase blocks which will be written next, known as the pool. The mount process then needs to examine the first 64 PEBs to find a "super block" which points to the mapping, read the mapping, and then read the first page of each PEB in the pool to find changes to the mapping. When the pool is close to exhaustion, a new copy of the mapping with a new list of pool PEBs is written out. This is clearly a little more complex, but puts a firm cap on the mount time and so ensures scalability to much larger devices. 

#### UBIFS — the trees

With UBIFS, all the filesystem content — inodes, data, and directory entries — is stored in nodes in various arbitrary Logical Erase Blocks, and the addresses of these blocks are stored in a single B-tree. This is similar in some ways to reiserfs (originally known as "treefs") and Btrfs, and contrasts with filesystems like f2fs, NILFS2 and ext3 where inodes, file data, and directory entries are all stored with quite different indexing structures. 

The key for lookup in this B-tree is 64 bits wide, formed from a 32-bit inode number, a three-bit node type, and a 29-bit offset (for file data) or hash value (for directory entries). This last field, combined with a 4KB block size used for indexing, limits the size of the largest file to two terabytes, probably the smallest limit in the filesystem. 

Nodes in this B-tree are, like other nodes, stored in whichever erase block happens to be convenient. They are also like other nodes in that they are not sized to align with any "basic block" size. Rather the size is chosen based on the fan-out ratio configured for the filesystem. The default fan-out is eight, meaning that each B-tree node contains eight keys and eight pointers to other nodes, resulting in a little under 200 bytes per node. 

Using small nodes means that fewer bytes need to be written when updating indexes. On the other hand, there are more levels in the tree so more reading is likely to be required to find a node. The ideal trade off will depend on the relative speeds of reads and writes. For flash storage that serves reads a lot faster than writes — which is not uncommon, but seemingly not universal — it is likely that this fan-out provides a good balance. If not, it is easy to choose a different fan-out when creating a filesystem. 

New nodes in the filesystem do not get included in the indexing B-tree immediately. Rather, their addresses are written to a journal, to which a few LEBs are dedicated. When the filesystem is mounted, this journal is scanned, the nodes are found, and based on the type and other information in the node header, they are merged into the indexing tree. This merging also happens periodically while the filesystem is active, so that the journal can be truncated. Those nodes that are not yet indexed are sometimes referred to as "buds" — a term which at first can be somewhat confusing. Fortunately the UBIFS code is sprinkled with some very good documentation so it wasn't too hard to discover that "buds" were nodes that would soon be "leaves" of the B-tree, but weren't yet — [quite an apt botanical joke](<http://lxr.linux.no/#linux+v3.6.6/fs/ubifs/journal.c#L23>). 

Much like f2fs, UBIFS keeps several erase blocks open for writes at the same time so that different sorts of data can be kept separate from each other, which, among other things, can improve cleaning performance. These open blocks are referred to as different "journal heads". UBIFS has one "garbage collection" head where the cleaner writes nodes that it moves — somewhat like the "COLD" sections in f2fs. There is also a "base" head where inodes, directory entries, and other non-data nodes are written — a bit like the "NODE" sections in f2fs. Finally, there are one or more "data" heads where file data is written, though the current code doesn't appear to actually allow the "or more" aspect of the design. 

The other tree that UBIFS maintains is used for keeping track of free space or, more precisely, how many active nodes there are in each erase block. This tree is a radix tree with a fan-out of four. So if you write the address of a particular LEB in base four (also known as radix-four), then each digit would correspond to one level in the tree, and its value indicates which child to follow to get down to the next level. 

> ![\[Radix tree diagram\]](https://static.lwn.net/images/2012/ubifs-radix-tree.png)

This tree is stored in a completely separate part of the device with its own set of logical erase blocks, its own garbage collection, and consequently its own table of LEB usage counters. This last table must be small enough to fit in a single erase block and so imposes a (comfortably large) limit on the filesystem size. Keeping this tree separate seems like an odd decision, but doubtlessly simplifies the task of keeping track of device usage. If the node that records the usage of an LEB were to be stored in that LEB, there would be additional complexity which this approach avoids. 

#### A transition to FTL?

While JFFS2 clearly has limits, UBIFS seem to be much less limited. With 32 bits to address erase blocks which, themselves, could comfortably cover several megabytes, the addressing can scale to petabyte devices. The B-tree indexing scheme should allow large directories and large files to work just as well as small ones. The two terabyte limit on individual files might one day be a limit but that still seems a long way off. With the recent addition of fastmap for UBI, UBIFS would seem ready to scale to the biggest flash storage we have available. But it still requires raw flash access while a lot of flash devices force all access to pass through a flash translation layer. Could UBIFS still be useful on those devices? 

Given that the UBI layer looks a lot like an FTL it seems reasonable to wonder if UBI could be modified slightly to talk to a regular block device instead, and allow it to talk to an SD card or similar. Could this provide useful performance? 

Unfortunately such a conversion would be a little bit more than an afternoon's project. It would require: 

  * Changing the expectation that all I/O is synchronous. This might be as simple as waiting immediately after submitting each request, but it would be better if true multi-threading could be achieved. Currently, UBIFS disables readahead because it is incompatible with a synchronous I/O interface. 

  * Changing the expectation that byte-aligned reads are possible. UBIFS currently reads from a byte-aligned offset into a buffer, then decompresses from there. To work with the block layer it would be better to use a larger buffer that was sector-aligned, and then understand that the node read in would be found at an offset into that buffer, not at the beginning. 

  * Changing the expectation that erased blocks read as all ones. When mounting a filesystem, UBIFS scans various erase blocks and assumes anything that isn't `0xFF` is valid data. An FTL-based flash store will not provide that guarantee, so UBIFS would need to use a different mechanism to reliably detect dead data. This is not conceptually difficult but could be quite intrusive to the code. 

  * Finding some way to achieve the same effect as the atomic LEB updates that UBI can provide. Again, a well understood problem, but possibly intrusive to fix. 

So without a weekend to spare, that approach cannot be experimented with. Fortunately there is an alternative. As mentioned, there already exists a "block2mtd" driver which can be used to connect UBIFS, via UBI and mtd, to a block device. This driver in deliberately very simple and consequently quite inefficient. For example, it handles the `mtd_erase()` function by writing blocks full of `0xFF` to the device. However, it turns out that it is only an afternoons project to modify it to allow for credible testing. 

This [patch](<https://lwn.net/Articles/528623/>) modifies the block2mtd driver to handle `mtd_erase()` by recording the location of erased blocks in memory, return `0xFF` for any read of an erased block, and not write out the PEB headers until real data is to be written to the PEB.  The result of these changes is that the pattern of reads and, more importantly, writes to the block device will be much the same as the pattern of reads and writes expected from a more properly modified UBIFS. It is clearly not useful for real usage as important information is kept in memory, but it can provide a credible base for performance testing. 

The obvious choice of what to test it against is f2fs. Having examined the internals of both f2fs and UBIFS, we have found substantial similarity which is hardly surprising as they have both been designed to work with flash storage. Both write whole erase blocks at a time where possible, both have several erase blocks "open" at once, and both make some efforts to collect similar data into the same erase blocks. There are of course differences though: UBIFS probably scales better to large directories, it can compress data being written, and it does not currently support exporting via NFS, partly because of the difficulty of providing a stable index for directory entries. 

The compression support is probably most interesting. If the CPU is fast enough, compression might be faster than writing to flash and this could give UBIFS an edge in speed. 

I performed some testing with f2fs and UBIFS; the latter was tested twice, with and without the use of compression (the non-compression case is marked below as "NC"). Just for interest's sake I've added NILFS2, ext4 and Btrfs. None of these are particularly designed for FTL based flash, though NILFS2 can align writes with the erase blocks and so might perform well. The results of the last two should be treated very cautiously. No effort was made to tune them to the device used, and all the results are based on writing to an empty device. For f2fs, UBIFS, and NILFS2 we know that they can "clean" the device so they always write to unused erase blocks. ext4 and Btrfs do not do the same cleaning so it is quite possible that the performance will degrade on a more "aged" filesystem. So the real long term values for these filesystems might be better, and might be worse, than what we see here. 

For testing I used a new class 10 16GB microSD card, which claims 10MB/s throughput and seems to provide close to that for sequential IO. According to the [flashbench](<https://launchpad.net/flashbench>) tool, the card appears to have an 8MB erase block size; five erase blocks can be open at a time, and only the first erase block optimized for a PC-style file attribute table. The kernel used was 3.6.6 for openSUSE with the above mentioned patch and the [v3 release](<http://lkml.org/lkml/2012/10/31/156>) of f2fs. 

The tests performed were very simple. To measure small file performance, a tar archive of the Linux kernel (v3.7-rc6) was unpacked ten times and then — after unmounting and remounting — the files were read back in again and "`du`" and "`rm -r`" were timed to check metadata performance. The "`rm -r`" test was performed with a warm cache, immediately after the "`du -a`", which was performed on a cold cache. The average times in seconds for these operations were: 

> | ubifs| ubifs — NC| f2fs| NILFS2| ext4| Btrfs  
> ---|---|---|---|---|---|---  
> **Write kernel**|  72.4| 139.9| 118.4| 140.0| 135.5| 93.6  
> **Read kernel**|  72.5| 129.6| 175.7| 95.6| 108.8| 121.0  
> **`du -s`**|  9.9| 8.7| 48.6| 4.4| 4.4| 13.8  
> **`rm -r`**|  0.48| 0.45| 0.36| 11.0| 4.9| 33.6  
  
Some observations: 

  * UBIFS, with compression, is clearly the winner at reading and writing small files. This test was run on an Intel Core i7 processor running at 1GHz; on a slower processor, the effect might not be as big. Without compression, UBIFS is nearly the slowest, which is a little surprising, but that could be due to the multiple levels that data passes though (UBI, MTD, block2mtd). 

  * f2fs is surprisingly poor at simple metadata access (`du -s`). It is unlikely that this is due to the format chosen for the filesystem — the indirection of the Node Address Table is the only aspect of the design that could possibly cause this slowdown and it could explain at most a factor of two. This poor performance is probably some simple implementation issue. The number is stable across the ten runs, so it isn't just a fluke. 

  * Btrfs is surprisingly fast at writing. The kernel source tree is about 500MB in size, so this is around 5.5MB/sec, which is well below what the device can handle but is still faster than anything else. This presumably reflects the performance-tuning efforts that the Btrfs team have made. 

  * "`rm -r`" is surprisingly slow for the non-flash-focused filesystems, particularly Btrfs. The variance is high too. For ext4, the slowest "`rm -r`" took 32.4 seconds, while, for Btrfs, the slowest was 137.8 seconds — over 2 minutes. This seems to be one area where tuning the design for flash can be a big win. 

So there is little here to really encourage spending that weekend to make UBIFS work well directly on flash. Except for the compression advantage, we are unlikely to do much better than f2fs, which can be used without that weekend of work. We would at least need to see how compression performs on the processor found in the target device before focusing too much on it. 

As well as small files, I did some even simpler large-file tests. For this, I wrote and subsequently read two large, already compressed, files. One was an mp4 file with about one hour of video. The other was an openSUSE 12.2 install ISO image. Together they total about 6GB. The total times for each filesystem were: 

> | ubifs| ubifs — NC| f2fs| NILFS2| ext4| Btrfs  
> ---|---|---|---|---|---|---  
> write files|  850|  876|  838| 1522|  696| 863  
> read files |  1684|  1539|  571| 574|  571| 613  
  
The conclusions here are a bit different: 

  * Now ext4 is a clear winner on writes. It would be very interesting to work out why. The time translates to about 8.8MB/sec which is getting close to the theoretical maximum of 10MB/sec. 

  * Conversely, NILFS2 is a clear loser, taking nearly twice as long as the other filesystems. Two separate runs showed similar results so it looks like there is room for some performance tuning here. 

  * UBIFS is a clear loser on reads. This is probably because nodes are not aligned to sectors so some extra reading and extra copying is needed. 

  * The ability for UBIFS to compress data clearly doesn't help with these large files. UBIFS did a little better with compression enabled, suggesting that the files were partly compressible, but it wasn't enough to come close to f2fs. 

In summary, while f2fs appears to have room for improvement in some aspects of the implementation, there seems little benefit to be gained from pushing UBIFS into the arena of FTL-based devices. It will likely remain the best filesystem for raw flash, while f2fs certainly has some chance of positioning itself as the best filesystem for FTL-based flash. However, we certainly shouldn't write off ext4 or Btrfs. As noted earlier, these tests are not expected to give a firm picture of these two filesystems so we cannot read anything conclusive from them. However, it appears that both have something to offer, if only we can find a way to isolate that something.
