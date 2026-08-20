---
title: Btrfs send/receive
url: https://lwn.net/Articles/506244/
date: "July 11, 2012"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet July 11, 2012"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
July 11, 2012

The btrfs snapshot capability allows a system administrator to quickly capture the state of a filesystem at any given time. Thanks to the copy-on-write mechanism used by btrfs, snapshots share data with other snapshots or the "live" system; blocks are only duplicated when they are changed. While btrfs makes the creation and management of snapshots easy, it currently lacks the ability to efficiently determine what the differences are between two snapshots and save that information for future use. Given that some other advanced filesystems (ZFS, for example) offer that capability, btrfs can arguably be seen as falling a little short in this particular area. 

Happily, that situation appears to be about to change, as Alexander Block's [btrfs send/receive patch set](<https://lwn.net/Articles/505111/>) has been well received by the development community. In short, with this patch set (and the associated [user space tools](<https://lwn.net/Articles/506278/>)), btrfs can be instructed to calculate the set of changes made between two snapshots and serialize them to a file. That file can then be replayed elsewhere, possibly at some future time, to regenerate one snapshot from the other. 

This functionality is implemented with the new `BTRFS_IOC_SEND` `ioctl()` command. In its simplest form, this operation accepts a file descriptor representing a mounted volume and the subvolume ID corresponding to the snapshot of interest; it will then find the changes between the snapshot and the "parent" snapshot it was generated from. There are more options, though: 

  * The operation can actually take a list of snapshot/subvolume IDs and generate a combined file for all of them. 

  * The parent snapshot can be specified explicitly. That may be required for older btrfs volumes that lack the needed identifying information. It may also be useful to generate differences that skip over a set of snapshots — differences from a grandparent, say, instead of the direct parent. 

  * The command also accepts an optional list of "clone sources." Those are subvolumes that can be expected to exist on the receiving side; when possible, data blocks will be "cloned" from those snapshots rather than being written into the differences file. That reduces the size of the differences, and enables better data sharing on the receive side. 

The generated file is essentially a set of instructions for converting the parent snapshot into the one being "sent." The list of commands is surprisingly long, including operations like create a file (or directory, device node, FIFO, symbolic link, ...), rename or link a file, unlink a file, set and remove extended attributes, write data, clone data blocks, truncate a file, change ownership and permissions, set file times, and so on. The code that generates this file is also surprisingly long, being several thousand lines of complex, nearly uncommented functions (some of the comments that do exist, saying things like "magic happens here," are not entirely helpful). 

Interestingly, according to the patch introduction, the custom file format was not in the original plan. Instead, the output was meant to be in something close to the tar file format — close enough that the tar command could be used to extract data from it. Tar turned out not to have the needed capabilities, though, so a new format was created. The format should be considered to be in flux still, though, clearly, it will need to stabilize before this feature can be considered ready for production use. As it happens, the playback of this file can be done almost entirely in user space, so there is no need for a `BTRFS_IOC_RECEIVE` operation. 

At the command level, using this feature can be as simple as: 

```
btrfs send _snapshot_
```

This will send the given `snapshot` (in its entirety) to the standard output stream. Writing the command as: 

```
btrfs send -i _oldsnap_ _snapshot_
```

will cause the creation of an incremental send containing just the differences from `oldsnap`. The `receive` command can be used to apply a file created by `btrfs send` to an existing filesystem. 

The primary use case for this feature (which is clearly patterned after the ZFS send/receive functionality) is backups in various forms. A `cron` job could easily send a snapshot to a remote server on a regular basis, maintaining a mirror of a filesystem there. The send files can simply be stored as backups; an entire volume can be sent as a full backup, while snapshots are easily sent as incrementals. With some additional tooling, the send/receive feature could develop into an advanced backup capability with low-level support from the underlying filesystem. 

That is for some time in the future, though; the feature is currently experimental, and Alexander warns potential users: 

If you use it for backups, you're taking big risks and may end up with unusable backups. Please do not only count on btrfs send/receive backups! 

That said, there seems to be a fair amount of interest in this feature (btrfs creator Chris Mason [described](<https://lwn.net/Articles/506280/>) it as ""just awesome""), so chances are it will be worked into reasonable shape relatively quickly. Then btrfs will have one more useful feature and one less reason to be concerned about comparisons with that other filesystem.
