---
title: "Bcache: Caching beyond just RAM"
url: https://lwn.net/Articles/394672/
date: "July 2, 2010"
category: "Block layer-Caching"
author: "July 2, 2010 By William Stearns and Kent Overstreet"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

July 2, 2010

By William Stearns and Kent Overstreet

Kent Overstreet has been working on [bcache](<http://lwn.net/Articles/394905/>), which is a Linux kernel module intended to improve the performance of block devices. Instead of using just memory to cache hard drives, he proposes to use one or more solid-state storage devices (SSDs) to cache block data (hence bcache, a **b** lock device **cache**). 

The code is largely filesystem agnostic as long as the filesystem has an embedded UUID (which includes the standard Linux filesystems and swap devices). When data is read from the hard drive, a copy is saved to the SSD. Later, when one of those sectors needs to be retrieved again, the kernel checks to see if it's still in page cache. If so, the read comes from RAM just like it always has on Linux. If it's not in RAM but it is on the SSD, it's read from there. It's like we've added 64GB or more of - slower - RAM to the system and devoted it to caching.

The design of bcache allows the use of more than one SSD to perform caching. It is also possible to cache more than one existing filesystem, or choose instead to just cache a small number of performance-critical filesystems. It would be perfectly reasonable to cache a partition used by a database manager, but not cache a large filesystem holding archives of old projects. The standard Linux page cache can be wiped out by copying a few large (near the size of your system RAM) files. Large file copies on that project archive partition won't wipe out an SSD-based cache using bcache.

Another potential use is using local media to cache remote disks. You could use an existing partition/drive/loopback device to cache any of the following: AOE (ATA-over-Ethernet) drive, SAN LUN, DRBD or NBD remote drives, iSCSI drives, local CD, or local (slow) USB thumb drives. The local cache wouldn't have to be an SSD, it would just have to be faster than the media you're caching.

Note that this type of caching is only for block devices (anything that shows up as a block device in `/dev/`). It isn't for network _filesystems_ like NFS, CIFS, and so on (see the [FS-cache module](<http://lwn.net/Articles/312708/>) for the ability to cache individual files on an NFS or AFS client).

#### Implementation

To intercept filesystem operations, bcache hooks into the top of the block layer, in `__generic_make_request()`. It thus works entirely in terms of BIO structures. By hooking into the sole function through which all disk requests pass, bcache doesn't need to make any changes to block device naming or filesystem mounting. If `/dev/md5` was originally mounted on `/usr/`, it continues to show up as `/dev/md5` mounted on `/usr/` after bcache is enabled for it. Because the caching is transparent, there are no changes to the boot process; in fact, bcache could be turned on long after the system is up and running. This approach of intercepting bio requests in the background allows us to start and stop caching on the fly, to add and remove cache devices, and to boot with or without bcache.

bcache's design focuses on avoiding random writes and playing to SSD's strengths. Roughly, a cache device is divided up into buckets, which are intended to match the physical disk's erase blocks. Each bucket has an eight-bit generation number which is maintained in a separate array on the SSD just past the superblock. Pointers (both to btree buckets and to cached data) contain the generation number of the bucket they point to; thus to free and reuse a bucket, it is sufficient to increment the generation number.

This mechanism allows bcache keep the cache device completely full; when it wants to write some new data, it just picks a bucket, increments its generation number, invalidating all the existing pointers to it. Garbage collection will remove the actual pointers eventually; there is no need for backpointers or any other infrastructure.

For each bucket, bcache remembers the generation number from the last time it had a full garbage collection performed. Once the difference between the current generation number and the remembered number reaches 64, it's time for another garbage collection. Since the generation number has no chance to wrap, an 8-bit generation number is sufficient.

Unlike standard btrees, bcache's btrees aren't kept fully sorted, so if you want to insert a key you don't have to rebalance the whole thing. Rather, they're kept sorted according to when they were written out; if the first ten pages are already on disk, bcache will insert into the 11th page, in sorted order, until it's full. During garbage collection (and, in the future, during insertion if there are too many sets) it'll re-sort the whole bucket. This means bcache doesn't have much of the index pinned in memory, but it also doesn't have to do much work to keep the index written out. Compare that to a hash table of ten million or so entries and the advantages are obvious.

#### State of the code

Currently, the code is looking fairly stable; it's survived overnight torture tests. Production is still a ways away, and there are some corner cases and IO error handling to flesh out, but more testing would be very welcome at this point. 

There's a long list of planned features: 

**IO tracking** : By keeping a hash of the most recent IOs, it's possible to track sequential IO and bypass the cache - large file copies, backups, and raid resyncs will all bypass the cache. This one's mostly implemented. 

**Write behind caching** : Synchronous writes are becoming more and more of a problem for many workloads, but with bcache random writes become sequential writes. If you've got the ability to buffer 50 or 100GB of writes, many might never hit your RAID array before being rewritten. 

The initial write behind caching implementation isn't far off, but at first it'll work by flushing out new btree pages to disk quicker, before they fill up - journaling won't be required because the only metadata to write out in order is a single index. Since we have garbage collection, we can mark buckets as in-use before we use them and leaking free space is a non issue. (This is analogous to [soft updates](<http://lwn.net/Articles/339337/>) \- only much more practical). However, journaling would still be advantageous so that all new keys can be flushed out sequentially; then updates to the btree can happen as pages fill up, versus many smaller writes so that synchronous writes can be completed quickly. Since bcache's btree is very flat, this won't be much of an issue for most workloads, but should still be worthwhile. 

**Multiple cache devices** have been planned from the start, and mostly implemented. Suppose you had multiple SSDs to use - you could stripe them, but then you have no redundancy, which is a problem for writeback caching. Or you could mirror them, but then you're pointlessly duplicating data that's present elsewhere. Bcache will be able to mirror only the dirty data, and then drop one of the copies when it's flushed out. 

**Checksumming** is a ways off, but definitely planned; it'll keep checksums of all the cached data in the btree, analogous to what Btrfs does. If a checksum doesn't match, that data can be simply tossed, the error logged, and the data read from the backing device or redundant copy. 

There's also a lot of room for experimentation and potential improvement in the various heuristics. Right now the cache functions in a least-recently-used (LRU) mode, but it's flexible enough to allow for other schemes. Potentially, we can retain data based on how much real time it saves the backing device, calculated from both the seek time and bandwidth. 

#### Sample performance numbers

Of course, performance is the only reason to use bcache, so benchmarks matter. Unfortunately there's still an odd bug affecting buffered IO so the current benchmarks don't yet fully reflect bcache's potential, but are more a measure of current progress. [Bonnie](<http://www.coker.com.au/bonnie++/>) isn't particularly indicative of real world performance, but has the advantage of familiarity and being easy to interpret; here is the bonnie output: 

Uncached: SATA 2 TB Western Digital Green hard drive 

```
Version  1.96       ------Sequential Output------ --Sequential Input- --Random-
    Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
    Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
    utumno          16G   672  91 68156   7 36398   4  2837  98 102864   5 269.3   2
    Latency             14400us    2014ms   12486ms   18666us     549ms     460ms
```

And now cached with a 64 gb Corsair Nova: 

```
Version  1.96       ------Sequential Output------ --Sequential Input- --Random-
    Concurrency   1     -Per Chr- --Block-- -Rewrite- -Per Chr- --Block-- --Seeks--
    Machine        Size K/sec %CP K/sec %CP K/sec %CP K/sec %CP K/sec %CP  /sec %CP
    utumno          16G   536  92 70825   7 53352   7  2785  99 181433  11  1756  15
    Latency             14773us    1826ms    3153ms    3918us    2212us   12480us
```

In these numbers, the per character columns are mostly irrelevant for our purposes, as they're affected by other parts of the kernel. The write and rewrite numbers are only interesting in that they don't go down, since bcache isn't doing write behind caching yet. The sequential input is reading data bonnie previously wrote, and thus should all be coming from the SSD. That's where bcache is lacking, the SSD is capable of about 235 mb/sec. The random IO numbers are actually about 90% reads, 10% writes of 4k each; without write behind caching bonnie is actually bottlenecked on the writes hitting the spinning metal disk, and bcache isn't that far off from the theoretical maximum. 

#### For more information

The [bcache wiki](<http://bcache.evilpiepirate.org/>) holds more details about the software, more formal benchmark numbers, and sample commands for getting started. 

The git repository for the kernel code is available at `git://evilpiepirate.org/~kent/linux-bcache.git`. The userspace tools are in a separate repository: `git://evilpiepirate.org/~kent/bcache-tools.git`. Both are viewable with a web browser at the [gitweb site](<http://evilpiepirate.org/gitweb>).
