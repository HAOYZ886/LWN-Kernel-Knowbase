---
title: "Btrfs: Getting started"
url: https://lwn.net/Articles/577218/
date: "December 17, 2013"
category: "Btrfs-LWNs guide to; Filesystems-Btrfs"
author: "By Jonathan Corbet December 17, 2013 LWN's guide to Btrfs"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
December 17, 2013

* * *

[LWN's guide to Btrfs](<https://lwn.net/Articles/576276/>)

This is the second article in a series on the Btrfs filesystem; those who have not seen [the first segment](<https://lwn.net/Articles/576276/>) may wish to take a quick look. This installment will cover the basics of finding the requisite software and getting started with a Btrfs filesystem, while leaving the advanced features for the future. Using Btrfs as a simple Unix-style filesystem is a straightforward matter once the proper tools are in place. 

The Btrfs filesystem code itself has been in the mainline kernel since the 2.6.29 release in early 2009. Since then, development of the in-kernel code has mostly been done upstream, so the mainline kernel contains all of the code that is deemed ready for use. In general, users wanting to use Btrfs for real work are probably best advised to stay close to the current mainline releases. Fixes are still being made at a high rate; it is probably preferable to run the fixed code than to get a demonstration of why the fixes were necessary. One can get even newer code by pulling from the [Btrfs development repository](<https://btrfs.wiki.kernel.org/index.php/Btrfs_source_repositories>), but that may be a bit _too_ new for anybody who is not actively developing Btrfs. 

The current user-space tools, which handle the creation and management of Btrfs filesystems, can be pulled from the repository at: 

```
git://git.kernel.org/pub/scm/linux/kernel/git/mason/btrfs-progs.git
```

Until recently, the last "release" of btrfs-progs was 0.19, made in June 2009\. Toward the end of November, though, the version number [was set to "v3.12"](<https://lwn.net/Articles/577222/>), inaugurating a new era in which version numbering will be tied to kernel releases. Btrfs developer Chris Mason noted at the time that he expected to make btrfs-progs releases with approximately the same frequency as the kernel going forward. Since much of the needed work is on the user-space side, this should be a welcome development for Btrfs users. 

Once again, those wanting to make serious use of Btrfs are likely to want to run something close to the current versions of the supporting user-space utilities. A lot of work (and bug fixes) is going into this code, but one needs to stay current to take advantage of that work. Some distributions follow progress in the btrfs-progs repository more closely than others; Fedora 19 already has v3.12, for example, so there is no real need for Fedora users to build their own version. Users whose distribution does not track the btrfs-progs repository so closely may want to install their own version built from the repository. 

#### Creating and mounting Btrfs filesystems

The utility to create a Btrfs filesystem is, unsurprisingly, `mkfs.btrfs`; it can be invoked directly or via the `mkfs` program. In its simplest form, it can be run as: 

```
mkfs.btrfs /dev/partition-name
```

Where `partition-name` is, of course, the actual name of the partition that is to contain the filesystem. 

Naturally, `mkfs.btrfs` has a fair number of options, though fewer than some other filesystems offer. Some of those that are relevant for basic usage include `--force` (necessary to convince `mkfs.btrfs` to overwrite an existing filesystem on the target partition), `--label` to set a label, and `--version` to just print out the version number and exit. One can also specify `--mixed` to cause the filesystem to mix data and metadata blocks together. Normally that will slow things down, so it is only recommended for situations where space is at an absolute premium; the [man page](<http://man7.org/linux/man-pages/man8/mkfs.btrfs.8.html>) suggests only using it for filesystems up to 1GB in size. 

Btrfs filesystems are made accessible via the `mount` command as usual. Like most non-trivial filesystems, Btrfs has [a number of specialized mount options](<https://btrfs.wiki.kernel.org/index.php/Mount_options>) that can be used to control its behavior. Some of these options will be discussed in later installments; a few that are of general interest include: 

> `autodefrag`
>      Enables automatic defragmentation of the filesystem in the background while it is running. Comments in the documentation suggest that this feature is still under development and may not produce optimal results for all workloads. 
> 
> `compress [=zlib|lzo|no]`
>      Turn on compression of data. With an argument, it specifies which compression algorithm should be used. The `compress-force` option forces the use of compression even on files that do not compress well. 
> 
> `nodatacow`
>      Turns off the copy-on-write mechanism, but only for newly created files. Turning off COW removes an important integrity mechanism and disables compression and data checksumming. In a few situations (the documentation says "large database files") there may be a significant performance improvement, but most users will probably not want to use this option. 
> 
> `nodatasum`
>      Turns off the creation of data checksums for newly created files. 

A mounted Btrfs filesystem feels mostly like any other Linux filesystem. Every now and then, some differences leak out. It can be disconcerting, for example, to delete a large file and not see an increase in the amount of available free space. Look back a minute or two later, though, and the missing space will have reappeared — assuming, of course, that said large file does not exist in any snapshots. Btrfs does a lot more work in the background than many other filesystems do. 

#### Other Btrfs tools

The btrfs-progs repository contains a number of programs beyond `mkfs.btrfs`. One of the more recent additions is the `btrfsck` filesystem check and repair tool. The [man page](<http://man7.org/linux/man-pages/man8/btrfsck.8.html>) makes the newness of this tool clear: ""Considering it is not well-tested in real-life situations yet, if you have a broken Btrfs filesystem, `btrfsck` may not repair but cause additional damages."" So users will want to think hard before running `btrfsck` in the `--repair` mode and, probably, make use of the "restore" functionality described below. 

The lack of a battle-hardened `btrfsck` utility remains one of the top reasons why system administrators often shy away from this filesystem. But the sad truth is that the only way to really make a truly comprehensive filesystem repair tool is to observe, over time, the ways in which a filesystem can become corrupted and come up with ways to fix those problems. So `btrfsck` will eventually mature into a tool that can handle a wide variety of problems, but there are no easy ways to shortcut that process. 

Meanwhile, anybody working with Btrfs will eventually need to make use of another tool, called simply `btrfs`. This tool is the Swiss Army Knife of the Btrfs world; it can be used to perform a wide variety of actions on a Btrfs filesystem. Thus, unsurprisingly, `btrfs` implements a large number of commands, many of which will be examined in the later parts of this series. A few that merit mention now are: 

> `btrfs filesystem df _filesystem_`
>      Provides free space information about the given `_filesystem_` with more detail than is available from the standard `df` command. 
> 
> `btrfs filesystem show [_filesystem_]`
>      Print information about one or more of the available Btrfs filesystems. 
> 
> `btrfs filesystem defragment [_file..._]`
>      Perform online defragmentation of a Btrfs filesystem; defragmentation is limited to the given files if they are specified. 
> 
> `btrfs restore _device_`
>      This command will try to extract the data from the given `_device_` , which, presumably, contains a filesystem with problems. By using this tool prior to attempting to repair the filesystem with `btrfsck`, a system administrator can maximize the chances of retrieving the data from the device even if `btrfsck` fails badly. See [this wiki page](<https://btrfs.wiki.kernel.org/index.php/Restore>) for details on how to use this tool. 
> 
> `btrfs scrub _filesystem_`
>      Launch a "scrub" operation on the given filesystem; scrubbing involves checking metadata and data against the checksums stored in the filesystem and correcting any errors found. Scrubbing can take some time, needless to say; it can be paused and resumed with variants of the `btrfs scrub` command if need be. 
> 
> `btrfs send _subvol_`
> `btrfs receive _mount_`
>      Controls the [send/receive](<https://lwn.net/Articles/506244/>) functionality, which can be used to replicate filesystems remotely or to implement incremental backup operations. 

The basics described thus far are enough to get started with Btrfs, treating it as just another Unix-style filesystem, possibly with added compression and data checksumming. But it's the advanced features of the Btrfs filesystem that make it truly unique in the Linux world. One of those features — the built-in multiple-device and RAID functionality — will be the subject of the next installment in this series.
