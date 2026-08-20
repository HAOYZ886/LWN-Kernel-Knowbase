---
title: "LSFMM: Btrfs: \"are we there yet?\""
url: https://lwn.net/Articles/548937/
date: "May 1, 2013"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jake Edge May 1, 2013 LSFMM Summit 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
May 1, 2013

* * *

[LSFMM Summit 2013](<https://lwn.net/Articles/LSFMM2013/>)

At the 2013 LSFMM Summit, Btrfs developer Josef Bacik gave an update on the status of the filesystem. It has seen a lot of changes over the last year, he said, with around 800 changesets being merged. He contrasted that with ext4, which has seen around 250. There are a number of features being worked on, including [subvolume quota groups [PDF]](<http://sensille.com/qgroups.pdf>) and a new [restriper](<https://lwn.net/Articles/474790/>) to use when adding more disks to a RAID filesystem. 

The performance of Btrfs is now roughly the same as ext4 and twice that of XFS on spinning disks, Bacik said. On Fusion-io devices it is "abysmally slow", about the same as XFS. That is caused by the way that `fsync()` works--"write wait, write wait"--which he is hoping to get around by using [atomic writes](<https://lwn.net/Articles/548116/>). This is for workloads using direct I/O, which is awful for Btrfs. In particular, writing 4K then doing an `fsync()` is something of a worst case for Btrfs. 

The problems with handling full filesystems are "kind of an ongoing thing", Bacik said. There always seems to be something broken in that path. On the other hand, [send/receive](<https://lwn.net/Articles/506244/>) is working well, and defragmentation is working better. The extended inode references (IREF) problem has been fixed. That limited the number of hard links to a specific inode in a directory to two in the worst case, and only 40 in the best case. It is now only limited by disk space. 

RAID 5/6 has finally been merged, he said. It is not power-failure-safe yet, though that fix is coming soon. It requires a format change, which has delayed it somewhat. The code for replacing a broken drive is "much cleaner and faster". It is also a lot easier for administrators to use. There is an `fsck`, now, that does work and fixes problems in the filesystem. It checks the extent trees and checksum trees along with the free space in the filesystem. There is also [btrfs-image](<https://btrfs.wiki.kernel.org/index.php/Btrfs-image>) tool for creating an image of a Btrfs partition. 

A new release of Btrfs will be coming soon, Bacik said. There will hopefully be more steady releases in the future, not just of the mainline code, but also the utilities in btrfsprogs. 

Running out of space (i.e. `ENOSPC`) has been a big problem for Btrfs, though he thinks there is now a solution for it. Basically, the filesystem never knows how much space metadata is going to take, so it seriously overestimates. The fix will be a special "chunk" in the log where any overflow goes. 

Online deduplication has gone through a couple of iterations. It will probably go into the 3.11 kernel, Bacik said. It will not be the default, and will require a format change before it can be turned on. Offline deduplication can be done in user space. 

In answer to the "are we there yet?" question, Bacik said that we would be by the "end of the year". He has said that for the last three years now, but is getting more comfortable that it really is stabilizing. In the past, he has never had time to work on features because he has been fixing bugs, but there are fewer bugs to fix now. There are also a bunch of user-space tools to help "if things go horribly wrong". The `ENOSPC` problem should be handled in the next few months. 

Toward the end of this year, or early next year, the project can start talking to distributions about becoming the default filesystem for new installations. Beyond that, performance is the next big focus for the team, he concluded.
