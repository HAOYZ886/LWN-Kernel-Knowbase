---
title: Filesystems for zoned block devices
url: https://lwn.net/Articles/788851/
date: "May 21, 2019"
category: "Block layer-Zoned devices; Filesystems"
author: "By Jake Edge May 21, 2019 LSFMM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 21, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

Damien Le Moal and Naohiro Aota led a combined storage and filesystem session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM) on filesystem work that has been done for zoned block devices. These devices have multiple zones with different characteristics; usually there are zones that can only be written in sequential order as well as conventional zones that can be written in random order. The genesis of zoned block devices is [shingled magnetic recording](<https://en.wikipedia.org/wiki/Shingled_magnetic_recording>) (SMR) devices, which were created to increase the capacity of hard disks, but at the cost of some flexibility. 

Le Moal began by noting that the session would not be about zoned block devices, as the "Zoned block devices and file systems" title might imply; it would instead focus on filesystems for zoned block devices. At this point, the only Linux filesystem with support for zoned devices is [F2FS](<https://en.wikipedia.org/wiki/F2FS>); that work is still ongoing as there are some bugs to fix and some features lacking. Work has also been done to [add support for Btrfs](<https://lwn.net/ml/linux-btrfs/20180809180450.5091-1-naota@elisp.net/>); he turned the mic over to Aota to talk about that. 

#### Btrfs

Getting Btrfs working on zoned block devices required aligning its internal "device extent" structure with the zones. If the extent size is smaller than any given zone, some space will be wasted; larger extents can cover multiple zones. Extents are allocated sequentially. Internal buffering has been added to sort write requests to maintain the sequential ordering required by the zone. 

[ ![\[Damien Le Moal & Naohiro Aota\]](https://static.lwn.net/images/2019/lsf-lemoal-aota-sm.jpg) ](<https://lwn.net/Articles/788849/>)

Multiple modes are supported for Btrfs, including single, DUP, RAID0, RAID1, and RAID10. There is no support for RAID5 or RAID6, Aota said, because larger SMR disks are not well suited for those RAID modes due to the long rebuild time required when drives fail. Le Moal added that those modes could be handled, but for 15TB drives, for example, the rebuild time will be extremely long. 

Aota said there are two missing features that are being worked on. "Device replace" is mostly done, but there are still some race conditions to iron out. Supporting [`fallocate()`](<http://man7.org/linux/man-pages/man2/fallocate.2.html>) has not been done yet; there are difficulties preallocating space in a sequential zone. Some kind of in-memory preallocation is what he is planning. Chris Mason did not think `fallocate()` support was important for the initial versions of this code; it is not really a high-priority item for copy-on-write (CoW) filesystems. He did not think the code should be held up for that. 

Going forward, NVMe Zone Namespace (ZNS) support is planned, Aota said. In devices supporting ZNS, there will be no conventional zones supporting random writes at all. That means the superblock will need to be copy on write, so two zones will be reserved for the superblock and the filesystem will switch back and forth between them. 

Ric Wheeler asked about how long RAID rebuilds would take for RAID5/6. Le Moal said it could take a day or more. Wheeler did not think that was excessive, at least for RAID6, and said that there may be interest in having that RAID mode. The RAID6 rebuild I/O could be done at a lower priority, Wheeler said. But Mason thought that RAID5/6 support could wait until later; once again, he does not want to see these patches get hung up on that. Le Moal said they would send their patches soon. 

#### ZoneFS

ZoneFS is a new filesystem that exposes zoned block devices to users in the simplest possible way, Le Moal said. It exports each zone as a file under the mountpoint in two directories: `/conventional` for random-access zones or `/sequential` for sequential-only zones. Under those directories, the zones will be files that use the zone number as the file name. 

ZoneFS presents a fixed set of files that cannot be changed, removed, or renamed, and new files cannot be added. The only truncation operations (i.e. [`truncate()` and `ftruncate()`](<http://man7.org/linux/man-pages/man2/truncate64.2.html>)) supported for the sequential zones are those that specify a zero length; they will simply set the zone's write pointer back to the start of the zone. There will be no on-disk metadata for the filesystem; the write pointer location indicates the size of a sequential file. 

ZoneFS may not look "super useful", he said, so why was it created? Applications could get the same effect by opening the block device file directly, but application developers are not comfortable with that; he gets lots of email asking for something like ZoneFS. It works well for certain applications (e.g. [RocksDB](<https://rocksdb.org/>) and [LevelDB](<https://github.com/google/leveldb>)) that already use sequential data structures. It is also easy to integrate the ZoneFS files with these applications. 

Beyond that, ZoneFS can be used to support ZNS as well. Unlike the disk vendors, however, the NVMe people are saying that there may be a performance cost from relying on implicit open and close zone operations, as Ted Ts'o pointed out. That is going to make it interesting for filesystems like Btrfs that are trying to operate on both types of media but have not added explicit opens and closes based on what the SMR hard disk vendors have said along the way. 

Hearing no strong opposition to the idea, Le Moal said he would be sending ZoneFS patches around soon.
