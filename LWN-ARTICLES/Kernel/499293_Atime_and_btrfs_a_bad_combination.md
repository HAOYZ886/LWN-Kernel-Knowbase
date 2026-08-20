---
title: "Atime and btrfs: a bad combination?"
url: https://lwn.net/Articles/499293/
date: "May 31, 2012"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet May 31, 2012"
---

By **Jonathan Corbet**  
May 31, 2012

Unix and Unix-like systems have traditionally recorded the time of last access for each file in the system. This practice has fallen partially out of favor over the last decade for a simple reason: writing the last-accessed time ("atime") takes up a lot of I/O bandwidth when lots of files are being read; see [this article from 2007](<https://lwn.net/Articles/244829/>), for example. The worst of the atime-related problems have long since been mitigated by moving to the "relatime" mount option by default; relatime only updates atime a maximum of once per day for unchanging files. But now it seems that atime recording can be especially problematic with the btrfs filesystem, and relatime may not help much. 

One of the core design features of btrfs is its copy-on-write nature. Blocks on disk are never modified in place; instead, when it becomes necessary to commit a change, the affected block is copied and rewritten into a new location. Copy-on-write applies to metadata as well as data; if a file's metadata (such as its last-accessed time) is changed, the block containing that metadata will be copied to a new spot. So, on btrfs, an operation that reads a lot of files (creating a tar archive, say, or a recursive grep) can, through atime updates, cause the copying and rewriting of a lot of metadata blocks. 

Needless to say, performance is not improved this way, but that is not where the big problem comes in. As Alexander Block [pointed out](<https://lwn.net/Articles/499294/>), the real problem has to do with the interaction between atime, copy-on-write, and snapshots. 

Btrfs provides a fast snapshotting feature that can create a copy of the state of the filesystem at a specific time. When a snapshot is created, it shares all data and metadata with the "trunk" filesystem. Should a file be changed, the resulting copy-on-write operation separates the trunk from the snapshot, keeping both versions of the data available. So snapshots can be thought of as being nearly free as long as the filesystem remains relatively quiet. Snapshots will share data and metadata, so they do not require a lot of additional space. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Atime updates change the situation, though. If somebody takes a snapshot of a filesystem, then performs a recursive grep on that filesystem, the last-access time of every file touched may be updated. That, in turn, can cause copy-on-write operations on each file's inode structure, with the result that many or all of the inodes in the filesystem may be duplicated. That can increase the space consumption of the filesystem considerably; Alexander posted [an example](<https://lwn.net/Articles/499297/>) where a recursive grep caused 2.2GB of free space to disappear. That is a surprising result for what is meant to be a read-only operation. 

Once upon a time, when disk capacities were measured in megabytes, it was said that the only standard feature of Unix systems was the message of the day telling users to clean up their files. Atime was often used by harried system administrators trying to recover some disk space; they would scan for infrequently-accessed files and, depending on how desperate the situation was and how powerful their users were, either send lists of unused files to users or simply back those files up to tape and delete them. It is somewhat ironic that a feature meant (among other things) to help keep disk space free has now, on next-generation filesystems, become part of the problem. 

It's worth noting that the relatime option (which only updates atime once per day unless the file has been modified since the last atime update) is of little help here. It only takes one atime update to force an inode to be rewritten and unshared with any snapshots. So the fact that such updates are limited to one per day offers little in the way of consolation. 

Users are also unlikely to be consoled by one other aspect of the problem pointed out by Alexander: since reading data can consume space in the filesystem, read operations might fail with "no space available" errors on an overflowing filesystem. That may make it difficult or impossible to fix the problem by copying data out of a full filesystem. By the time that happens, a typical user could be forgiven for thinking that, perhaps, they don't need last-accessed time tracking at all. 

Along those lines, Alexander suggested that it might be a good idea to default to "noatime" (which turns off atime recording entirely) for btrfs mounts, even if that means that btrfs would then behave differently than other filesystems. That idea was quickly shot down for a simple reason: there are still applications that actually need the atime information to function correctly. The classic example is the mutt email client which, in the absence of atime, cannot tell whether a mailbox contains unread mail. Programs that clean up temporary directories (tmpreaper or tmpwatch, for example) will fail without atime. There are also hierarchical storage systems that, like the Unix system administrator of old, use atime to determine when to move files to slower storage. So atime needs to stick around, lest users run into a different kind of unpleasant surprise. 

For now, the only recourse for users who run into (or are worried about) this problem is to explicitly mount their filesystems with the "noatime" option. Further ahead, it might be possible to make some tweaks to btrfs to mitigate the problem; Boaz Harrosh [suggested](<https://lwn.net/Articles/499304/>) disabling atime updates when the free space falls below a certain threshold or moving the atime data into a separate data structure. But nobody appears to be working on such solutions now. So it may be that, as usage of btrfs grows, users will occasionally be surprised that reading a file can consume space in their filesystems.
