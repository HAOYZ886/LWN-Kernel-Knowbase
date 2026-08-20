---
title: Copy offload
url: https://lwn.net/Articles/637436/
date: "March 25, 2015"
category: Block layer
author: "By Jake Edge March 25, 2015 LSFMM 2015"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
March 25, 2015

* * *

[LSFMM 2015](<https://lwn.net/Articles/lsfmm2015/>)

In the final combined storage-and-filesystem session at the 2015 LSFMM Summit, Zach Brown and Martin Petersen teamed up to describe the state of and plans for supporting copy offload, which is a way of handing the work of copying a file to a filesystem or lower-level storage device, where the task can often be optimized. The functionality has been available in storage devices for eight years or so, Brown said. 

The current strategy is to add a new system call, `copy_file_range()`, that takes two file descriptors with pointers to offsets and lengths, Brown said. As the later discussion indicated, those file descriptors could be for files on different filesystems, but some feel that they should be restricted to a single filesystem. The big difference from earlier proposals is that callers are now required to create the destination file. That avoids some race conditions in the virtual filesystem (VFS) layer. 

[ ![\[Zach Brown\]](https://static.lwn.net/images/2015/lsf-brown-sm.jpg) ](<https://lwn.net/Articles/637440/>)

The remaining contentious parts for the system call are minor, he continued. For example, a flag value for the length could indicate that the entire source file should be copied. There is a "whole world of shit we can argue about", he said, since there are 32 bits worth of flag values available. The contentious piece is on the block side, he said. Petersen has added support, but the device mapper developers did not like the approach he took. 

For Btrfs, the system call is a wrapper around the existing `ioctl()`, though there are some alignment issues still to be worked out. Chris Mason said that for Btrfs there are different options for doing copy offload. Creating a directory subvolume is a constant-time operation that can make a copy of an entire file (using copy on write or COW). Making a file copy directly, which could support a range in the file (again, using COW), is proportional to the number of extents in the file. Brown suggested that under the covers Btrfs could implement the copy as a subvolume creation if the copy is for a whole file. 

Ric Wheeler seemed to sum up the feeling of many when he said that "anything that works is better than years of nothing" for copy-offload support. 

Petersen said that SCSI support for copy offload has advanced since last year, even though he had said it was done then. It now supports more features. There are some patches that add copy-offload support to the device mapper [kcopyd](<https://www.kernel.org/doc/Documentation/device-mapper/kcopyd.txt>) (dm-kcopyd), though he "did not agree with the approach exactly". He has also added support for token-based copy offload, where device-generated tokens are used to identify the data of interest at the storage level. The block and SCSI support for copy offload has just been waiting for a user other than dm-kcopyd, he said. 

[ ![\[Martin Petersen\]](https://static.lwn.net/images/2015/lsf-petersen-sm.jpg) ](<https://lwn.net/Articles/637441/>)

Brown noted that callers of `copy_file_range()` could perhaps get an error return if the underlying storage did not support copy offload. That way the caller could decide whether to fall back to a regular copy or not. A flag could be added to the call to do that fallback in the kernel, too. 

The new system call would allow copying between files between two different mounted filesystems as long as both support copy offload, at least conceptually, but Christoph Hellwig thought that should be left for an add-on patch. All of the existing system calls will only work within a single mountpoint, he said, so making an exception needs to be considered carefully. Wheeler said that being able to do copies between mountpoints is a powerful feature, but Hellwig thought it should wait until someone actually needs that functionality and can provide a good implementation. It is never a problem to relax restrictions on system calls, Hellwig said. 

The cross-filesystem copying feature is most important for network filesystems, Hellwig said. Wheeler disagreed, saying that it is also important for local filesystems. Hellwig said there needs to be a well-thought-out interface, so that users don't get locked into `ioctl()`-based mechanisms. Block-based filesystems could defer to the lower-level copy-offload support, he suggested. There is "more than one way to skin the cat; we just have to find a cat that we can skin", Dave Chinner said with a chuckle. 

Step one should be to get the single-mountpoint system call implementation in, Hellwig said. Getting the block-layer support in should be step two. "Anything more fancy can follow". He also thought that token-based copies "make zero sense" from a user-interface perspective. That should be hidden in the lower levels. Finally, there should be an asynchronous interface with a notification when the operation completes. 

The sense in the room was that copy-offload support is nearing inclusion after being discussed for several years at LSFMM. We will have to wait and see what gets into the mainline or whether copy offload will be on the agenda at next year's summit in Raleigh, North Carolina. 

[I would like to thank the Linux Foundation for travel support to Boston for the summit.]
