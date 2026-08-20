---
title: btrfs and NILFS
url: https://lwn.net/Articles/238923/
date: "June 19, 2007"
category: "Btrfs; Filesystems-Btrfs; NILFS"
author: 
---

Almost exactly one year ago, as the developers were discussing changes to the venerable ext3 filesystem, Andrew Morton was [heard to say](<http://lwn.net/Articles/187346/>): 

All that being said, Linux's filesystems are looking increasingly crufty and we are getting to the time where we would benefit from a greenfield start-a-new-one. That new one might even be based on reiser4 - has anyone looked? It's been sitting around for a couple of years. 

Reiser4 looks like it may continue to sit around for a while yet. But that does not mean that there is no interest in the creation of interesting new filesystems. LogFS was [discussed here](<http://lwn.net/Articles/234441/>) in May, but it's not the only newcomer in the filesystem arena. 

The most interesting new contender, perhaps, is btrfs, which was [announced](<http://lwn.net/Articles/237904/>) by Chris Mason on June 12\. It is an entirely new filesystem intended for standard rotating storage with a number of interesting features. These include: 

  * Btrfs is a fully extent-based filesystem, meaning that it can store large files far more efficiently than ext3 (the in-development ext4 filesystem has extent support). An extent-based filesystem does away with the long lists of pointers to the individual blocks contained within a file; instead, groups of contiguous blocks ("extents") are tracked together. The result is far less metadata overhead, especially with large files. For very small files, btrfs will store the file contents themselves within the extent structure, eliminating the need for a separate block allocation. 

  * Filesystems can be split into "subvolumes," each of which has its own directory structure and disk quota. Subvolumes can be used to subdivide a btrfs filesystem, but there is another interesting use of them... 

  * Btrfs can do snapshotting - freezing the state of the filesystem at any given time. Snapshots are just subvolumes; they become a separate, independent directory tree which can be navigated independently from the "live" filesystem. Interestingly, though, btrfs snapshots are also live, and can be modified after being taken and snapshotted as well. 

  * Supporting subvolumes and snapshots forces a copy-on-write structure onto btrfs. If a given extent is written to, it will be copied and the new data written to the copy. Extents have reference counts; creating a snapshot, for example, will cause reference counts to be incremented. When an extent contained in both a snapshot and the "real" filesystem is modified, it will be copied for whatever subvolume is being changed but will remain in place, unchanged in the other. If the snapshot is eventually removed, all associated reference counts will be decremented and any unused extents will be reclaimed. 

  * The subvolume and snapshot mechanism eliminates the need for a separate journaling feature. Changes to the filesystem can be made transactional simply by taking a snapshot which only lasts until the transaction completes. 

  * This filesystem checksums everything - data and metadata both. As a result, it is able to detect many types of filesystem corruption on the fly. 

Fast filesystem checking is also an important design goal for btrfs. The data and metadata are laid out in a way that allows the offline filesystem checker to read the disk in a nearly sequential manner. That should speed the process considerably; filesystem checking usually involves vast numbers of seek operations. Online filesystem checking is also in the plans, though it has not been implemented yet; once it is working, this feature could eliminate the need for separate, mount-time filesystem checks entirely. 

This filesystem is in a very early state - not recommended for data which one might actually want to keep. There's not been a whole lot of benchmarking done, and, presumably, a lot of optimization work still to happen. For example, the entire filesystem is currently protected by a single mutex, a solution which is unlikely to perform well on those leading-edge 4096-processor systems. Little details - like not oopsing when the filesystem runs out of space, direct I/O, writing via `mmap()`, extended attributes, asynchronous I/O, and more - have yet to be taken care of. But btrfs has garnered a considerable amount of interest; if it lives up to its initial promise we could find ourselves using btrfs-based systems in the future. 

(For more information, see [the btrfs project page](<http://oss.oracle.com/projects/btrfs/>)). 

Another recently-announced filesystem is [NILFS](<http://lwn.net/Articles/238891/>), which is now at version 2.0. NILFS is a log-structured filesystem, in that the storage medium is treated like a circular buffer and new blocks are always written to the end. These filesystems tend to do very well on benchmarks which measure write performance, since all writes go to a contiguous set of blocks; read performance is not always quite as good. Log-structured filesystems are often used for flash media since they will naturally perform wear-leveling; it would appear, however, that NILFS is not aimed at flash devices. 

Instead, NILFS emphasizes snapshots. The log-structured approach is a specific form of copy-on-write behavior, so it naturally lends itself to the creation of filesystem snapshots. The NILFS developers [talk](<https://lwn.net/Articles/238928/>) about the creation of "continuous snapshots" which can be used to recover from user-initiated filesystem problems - those of the "`rm -r`" variety. NILFS claims scalability through 64-bit data structures, but, interestingly, support for the x86_64 architecture remains on the "TODO list." The filesystem does not yet have support for extents. 

More information on NILFS can be found on [nilfs.org](<http://www.nilfs.org/en/index.html>).
