---
title: Preventing information leaks from ext4 filesystems
url: https://lwn.net/Articles/854054/
date: "April 27, 2021"
category: "Filesystems-ext4"
author: "By Jonathan Corbet April 27, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
April 27, 2021

A filesystem's role is to store information and retrieve it in its original form on request. But filesystems are also expected to prevent the retrieval of information by people who should not see it. That requirement extends to data that has been deleted; users expect that data to be truly gone and will not welcome its reappearance in surprising places. Some work being done with ext4 shows the kind of measures that are required to live up to that expectation. 

In early April, Leah Rumancik posted [a two-patch series](<https://lwn.net/ml/linux-ext4/20210407154202.1527941-1-leah.rumancik@gmail.com/>) making a couple of small changes to the ext4 filesystem implementation. The first of those caused the filesystem to, after a file is deleted, overwrite the space (on disk) where that file's name was stored. In response to a question about why this was needed, ext4 maintainer Ted Ts'o [explained](<https://lwn.net/ml/linux-ext4/YG59GE+8bhtVLOQr@mit.edu/>) that it was meant to deal with the case where users were storing personally identifiable information (PII) in the names of files. When a file of that nature is removed, the user would like to be sure that the PII is no longer stored on the disk; that means wiping out the file names as well. 

Dave Chinner quickly [objected](<https://lwn.net/ml/linux-ext4/20210408052155.GK1990290@dread.disaster.area/>) to this explanation. The real problem, he argued, is that users are storing PII as clear text; wiping directory entries is not a real solution to that problem: 

> This sounds more and more like "Don't encode PII in clear text anywhere" is a best practice that should be enforced with a big hammer. Filenames get everywhere and there's no easy way to prevent that because path lookups can be done by anyone in the kernel. This so much sounds like you're starting a game of whack-a-mole that can never be won. 
> 
> From a security perspective, this is just bad design. Storing PII in clear text filenames pretty much guarantees that the PII will leak because it can't be scrubbed/contained within application controlled boundaries. Trying to contain the spread of filenames within random kernel subsystems sounds like a fool's errand to me, especially given how few kernel developers will even know that filenames are considered sensitive information from a security perspective... 

The problem with that approach, as Ts'o [explained](<https://lwn.net/ml/linux-ext4/YG9YqkHfslwAdh2%2F@mit.edu/>), is that the people involved may not even know what their ""legacy workloads"" are doing in this regard. Rather than risk the possibility of exposing PII that nobody even knew was there, he said, it is better to simply clear the file names. 

Of course, if a file's name constitutes PII, its contents are likely to be interesting as well. It is possible, though expensive, to overwrite the data in files when they are deleted. An alternative, on modern storage devices at least, is to use the [`FITRIM` `ioctl()` command](<https://lwn.net/Articles/417809/>) to tell the drive to discard the data. Even if this operation does not physically erase that data, it should make the data inaccessible afterward. For this reason (and others), administrators who are concerned about the persistence of deleted data tend to arrange for `FITRIM` operations to be run regularly. 

There is one little remaining issue, though, for the truly paranoid. Ext4, like many contemporary filesystems, uses journaling to ensure that the filesystem remains consistent even if the system is interrupted at an inopportune time. To implement that sort of robustness, metadata written to ext4 filesystems is also written to the journal (and file data can optionally be written there too), so the possibility exists that the journal will contain PII even after all traces of it have been removed from the rest of the filesystem. A bad actor who gains access to the disk could harvest that data from the journal, even though it had already been deleted from the filesystem. 

To address this problem, Rumancik's second patch adds a new `ioctl()` command, called `FS_IOC_CHKPT_JRNL`, that forces the journal to be flushed out to persistent storage. There is an optional flag, `CHKPT_JRNL_DISCARD`, which causes the filesystem to run the equivalent of a `FITRIM` operation on the journal itself once the flush is complete. That ensures that no PII remains in the journal itself. 

Chinner (in the same email linked above) suggested that this behavior should be an option on the `FITRIM` operation rather than a separate command — before noting that `FITRIM` has no "flags" field and, thus, cannot be extended. Perhaps, he suggested, it is time for a separate `fstrim()` system call that could also trim the journal on request. A separate system call would also, by default, make the functionality available to all filesystems rather than being an ext4-specific feature. 

The two patches together are intended to help administrators ensure that data that has been deleted from a filesystem truly disappears. It looks like they will be taking separate paths into the kernel from here, though. Rumancik recently posted [a new version of the file-name overwrite patch](<https://lwn.net/ml/linux-ext4/20210422180834.2242353-1-leah.rumancik@gmail.com/>) on its own, and Ts'o subsequently [applied it](<https://lwn.net/ml/linux-ext4/YIHknqxngB1sUdie@mit.edu/>). The journal side of the problem, Ts'o [said](<https://lwn.net/ml/linux-ext4/YIHVWySvaECveV4l@mit.edu/>), is going to require some more discussion. Eventually, though, one can expect solutions to both problems to find their way into the kernel. That will help prevent the accidental disclosure of sensitive information from ext4 filesystems, even if the user is storing it in ill-advised ways.
