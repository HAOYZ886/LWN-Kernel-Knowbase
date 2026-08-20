---
title: The Next3 filesystem
url: https://lwn.net/Articles/387231/
date: "May 11, 2010"
category: "Filesystems-ext3"
author: "By Jonathan Corbet May 11, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 11, 2010

The ext3 filesystem is tried and true, but it lacks a number of features deemed interesting by contemporary users. Snapshots - the ability to quickly capture the state of the filesystem at an arbitrary time - is at the top of many lists. It is currently possible to use the LVM snapshotting feature with ext3, but snapshots taken through LVM have some significant limitations. The [Next3 filesystem](<http://sourceforge.net/apps/mediawiki/next3/index.php?title=Main_Page>) offers an approach which might prove easier and more flexible: snapshots implemented directly in ext3. 

Next3 was developed by CTERA Networks, which has started shipping it on its [C200 network-attached storage device](<http://www.ctera.com/home/ctera-c200.html>). This code has also been posted on SourceForge and proposed for merging into the mainline kernel. The Next3 filesystem adds a simple snapshot feature to ext3 in ways which are (mostly) compatible with the existing on-disk format. It looks like a useful feature, but its path into the mainline looks to be longer than its implementers might have hoped. 

The Next3 filesystem is a new filesystem type - it's not just an addition to ext3. At its core, it works by creating a special, magic file to represent a snapshot of the filesystem. The files have the same apparent size as the storage volume as a whole, but they are sparse files, so they take almost no space at the outset. When a change is made to a block on disk, the filesystem must first check to see whether that block has been saved in the most recent snapshot already. If not, the affected block is moved over to the snapshot file, and a new block is allocated to replace it. Thus, over time, disk blocks migrate to the snapshot file as they are rewritten with new contents. 

Gaining read-only access to a snapshot is a simple matter of doing a loopback mount of the snapshot file as an ext2 filesystem. The snapshot file is sufficiently magic that any attempts to read blocks in the holes (which represent blocks that have not been changed since the snapshot was taken) will be satisfied from a later snapshot - which will have captured the contents of that block when it was eventually changed - or from the underlying storage device. Deleting a snapshot requires moving changed blocks into the previous snapshot, if it exists, because the deleted snapshot holds blocks which are logically part of the earlier snapshots. 

The changes to the ext3 on-disk format are minimal, to the point that a Next3 filesystem can be mounted by the ordinary ext3 code. If snapshots exist, though, ext3 cannot be allowed to modify the filesystem, lest the changed blocks fail to be saved in the snapshot. So, when snapshots exist on the filesystem, it will be marked with a feature flag which forces ext3 to mount the filesystem readonly. 

On the performance side, the news is said to be mostly good. Writes will take a little longer due to the need to move the old block to a snapshot file. The worst performance impact is seemingly on truncate operations; these may have to save a large number of blocks and can get a lot slower. It is also worth noting that the moving of modified blocks to the snapshot file will, over time, wreck the nice, contiguous on-disk format that ext3 tries so hard to create, with an unfortunate effect on streaming read performance. Files which must not be fragmented can be marked with a special flag which will cause blocks to be copied into the snapshot file rather than moved; that will slow writes further, but will keep the file contiguous on disk. 

Next3 developer Amir Goldstein requested relatively quick review of the patches because he is trying to finalize some of the on-disk formatting. The [answer](<https://lwn.net/Articles/387240/>) he got from Ted Ts'o was probably not quite what he was looking for: 

Ext4 is where new development takes place in the ext2/3/4 series. So enhancements such as Next3 will probably not be received with great welcome into ext3. 

Amir's response was that, while porting the patches to ext4 is on the "we'll get around to it someday" list, that port is not an easy thing to do. The biggest problem, apparently, is making the movement of blocks into the snapshot file work properly with ext4's extent-oriented format. Beyond that, Amir says, he's not actually trying to get the changes into ext3 - he wants to merge a separate filesystem called Next3 which happens to be mostly compatible with ext3. 

The "separate Next3" approach is unlikely to fly very far, though. As Ted [put it](<https://lwn.net/Articles/387241/>), ext2, ext3, and ext4 are really just different implementations of the same basic filesystem format; this format has never really been forked. Next3, as a separate filesystem, would be a fork of the format. The fact that Next3 has taken over some data structure fields which are used to different purpose in ext4 has not helped matters: 

The "ext" in ext2 stands for "extended", as in the "the second extended file system" for Linux. It perhaps would be better if we had used the term "extensible", since that's the main thing about ext2/3/4 that has given it so much staying power. We've been able to add, in very carefully backwards and forwards compatible way, new features to the file system format. This is why I object to why Next3 uses some fields that overlaps with ext4. It means that e2fsprogs, which supports _one_ and _only_ _one_ file system format, will now need to support two file system formats. And that's not something I want to do. 

The answer appears fairly clear: patches adding the snapshot feature might be welcome, but not as a fork of the ext3 filesystem. At a bare minimum, the filesystem format will have to be changed to avoid conflicts with ext4, but the real solution appears to be simply implementing the patches on top of ext4 instead of ext3. That is a fair amount of extra work which might have been avoided had the Next3 developers talked with the community prior to starting to code.
