---
title: "JLS2009: A Btrfs update"
url: https://lwn.net/Articles/358940/
date: "October 27, 2009"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet October 27, 2009"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
October 27, 2009

Conferences can be a good opportunity to catch up with the state of ongoing projects. Even a detailed reading of the relevant mailing lists will not always shed light on what the developers are planning to do next, but a public presentation can inspire them to set out what they have in mind. Chris Mason's Btrfs talk at the Japan Linux Symposium was a good example of such a talk. 

The Btrfs filesystem was merged for the 2.6.29 kernel, mostly as a way to encourage wider testing and development. It is certainly not meant for production use at this time. That said, there are people doing serious work on top of Btrfs; it is getting to where it is stable enough for daring users. Current Btrfs includes an all-caps warning in the `Kconfig` file stating that the disk format has not yet been stabilized; Chris is planning to remove that warning, perhaps for the 2.6.33 release. Btrfs, in other words, is progressing quickly. 

One relatively recent addition is full use of zlib compression. Online resizing and defragmentation are coming along nicely. There has also been some work aimed at making synchronous I/O operations work well. 

Defragmentation in Btrfs is easy: any specific file can be defragmented by simply reading it and writing it back. Since Btrfs is a copy-on-write filesystem, this rewrite will create a new copy of the file's data which will be as contiguous as the filesystem is able to make it. This approach can also be used to control the layout of files on the filesystem. As an experiment, Chris took a bunch of boot-tracing data from a Moblin system [![\[Chris Mason\]](https://static.lwn.net/images/conf/ks-jls-09/chris-mason-sm.jpg)](<https://lwn.net/Articles/358959/>) and analyzed it to figure out which files were accessed, and in which order. He then rewrote the files in question to put them all in the same part of the disk. The result was a halving of the I/O time during boot, resulting in a faster system initialization and smiles all around. 

Performance of synchronous operations has been an important issue over the last year. On filesystems like ext3, an `fsync()` call will flush out a lot of data which is not related to the actual file involved; that adds a significant performance penalty for `fsync()` use and discourages careful programming. Btrfs has improved the situation by creating an entirely separate Btree on each filesystem which is used for synchronous I/O operations. That tree is managed identically to, but separately from, the regular filesystem tree. When an `fsync()` call comes along, Btrfs can use this tree to only force out operations for the specific file involved. That gives a major performance win over ext3 and ext4. 

A further improvement would be the ability to write a set of files, then flush them all out in a single operation. Btrfs could do that, but there's no way in POSIX to tell the kernel to flush multiple files at once. Fixing that is likely to involve a new system call. 

Btrfs provides a number of features which are also available via the device mapper and MD subsystems; some people have wondered if this duplication of features makes sense. But there are some good reasons for it; Chris gave a couple of examples: 

  * Doing snapshots at the device mapper/LVM layer involves making a lot more copies of the relevant data. Chris ran an experiment where he created a 400MB file, created a bunch of snapshots, then overwrote the file. Btrfs is able to just write the new version, while allowing all of the snapshots to share the old copy. LVM, instead, copies the data once for each snapshot. So this test, which ran in less than two seconds on Btrfs, took about ten _minutes_ with LVM. 

  * Anybody who has had to replace a drive in a RAID array knows that the rebuild process can be long and painful. While all of that data is being copied, the array runs slowly and does not provide the usual protections. The advantage of running RAID within Btrfs is that the filesystem knows which blocks contain useful data and which do not. So, while an MD-based RAID array must copy an entire drive's worth of data, Btrfs can get by without copying unused blocks. 

So what does the future hold? Chris says that the 2.6.32 kernel will include a version of Btrfs which is stable enough for early adopters to play with. In 2.6.33, with any luck, the filesystem will have RAID4 and RAID5 support. Things will then stabilize further for 2.6.34. Chris was typically cagey when talking about production use, though, pointing out that it always takes a number of years to develop complete confidence in a new filesystem. So, while those of us with curiosity, courage, and good backups could maybe be making regular use of Btrfs within a year, widespread adoption is likely to be rather farther away than that.
