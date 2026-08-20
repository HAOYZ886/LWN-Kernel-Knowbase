---
title: New APIs for filesystems
url: https://lwn.net/Articles/975444/
date: "May 30, 2024"
category: "Filesystems-APIs"
author: "By Jake Edge May 30, 2024 LSFMM+BPF"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
May 30, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

A discussion of extensions to the [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) system call comes up frequently at the [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>); this year's edition was no exception. Kent Overstreet led the first filesystem-only session at the summit on querying information about filesystems that have subvolumes and snapshots. While it was billed as a discussion on `statx()` additions, it ranged more widely over new APIs needed for modern filesystems. 

#### Brainstorming

Overstreet began the session with the idea that it would be something of a brainstorming exercise to come up with additions to the filesystem APIs. He had some thoughts on new features, but wanted to hear what other attendees were thinking so that a list of tasks could be gathered. He said that he did not plan to do all of the work on that list himself, but he would help coordinate it. 

He has started thinking about per-subvolume disk-accounting for bcachefs, which led him to the need for a way to iterate over subvolumes. He mentioned some previous discussion where Al Viro had an idea for an iterator API that would return a file descriptor for each open subvolume. "That was crazy and cool", Overstreet said; it also fits well with various [`openat()`](<https://man7.org/linux/man-pages/man2/open.2.html>)-style interfaces. He thinks there is a simpler approach, however. 

[ ![\[Kent Overstreet\]](https://static.lwn.net/images/2024/lsfmb-overstreet-sm.png) ](<https://lwn.net/Articles/975449/>)

Adding a flags parameter to [`opendir()`](<https://man7.org/linux/man-pages/man3/opendir.3.html>) would allow creating a flag for iterating over subvolumes and submounts. Subvolumes and mounts have a lot in common, he has noticed recently; user-space developers would like to have ways to work with them, which this would provide. 

Extended attributes (xattrs) on files are also in need of an iterator interface of some kind, he said. Those could smoothly fit into the scheme he is proposing. The existing [`getdents()`](<https://man7.org/linux/man-pages/man2/getdents64.2.html>) interface is "nice and clean", he said, so it could be used for xattrs as well. 

The `stx_subvol` field has recently been added to `statx()` for subvolume identifiers. Another flag for `statx()` will be needed to identify whether the file is in a snapshot. That way, coreutils would be able to, by default, filter out snapshots. That way, when someone is working through the filesystem, they do not see the same files over and over again. 

Steve French asked a "beginner question" about how to list the snapshots for a given mount in a generic fashion on Linux. Overstreet said that a snapshot is a type of subvolume and that "a subvolume is a fancy directory". This new `opendir()` interface could be used to iterate over the subvolumes and the new `statx()` flag could be used to check which are snapshots. 

All of the information that [`statfs()`](<https://man7.org/linux/man-pages/man2/statfs.2.html>) returns for a mounted filesystem should also be available for subvolumes, he said, "continuing with the theme that subvolumes and mount points actually have a lot in common". That includes things like disk-space usage and the number of inodes used. 

Dave Chinner said that XFS already has a similar interface based on project IDs, where a directory entry that corresponds to a particular project can be passed to `statfs()` to retrieve the information specific to that project. He said that filesystems could examine the passed-in directory and decide what to report based on that, so no new system call would be needed. Overstreet was skeptical that users who type `df` in their home directory would expect to only get information for the subvolume it is in, rather than the whole disk, as they do now. He thought a new system call would be the right way to approach it. 

French said that other operating systems have a way to simply open a version of a file from a snapshot without actually having to directly work with the entire snapshot subvolume itself. A user can simply open a file from a given snapshot identifier, which is convenient and not really possible on Linux. Overstreet acknowledged the problem, but said that he did not think a new system call was needed to support that use case. Using the new interfaces that are being discussed, user space should be able to handle that functionality, perhaps using read-only mounts of snapshots in such a way that the user does not directly have to work with them. 

#### User-space concerns

But Lennart Poettering said: "as a user-space person, I find it a bit weird" that `opendir()` is seen as a good interface for this functionality. In many ways, he finds `opendir()` to be "a terrible API" because it gives you a filename, but then you have to open the file to get more information, which does not necessarily match up because there can be a race between the two operations. He would much prefer to get a file descriptor when enumerating things so that the state cannot change between the two. 

There are some other mismatches between `opendir()` and subvolumes, he continued. Right now, user space expects to get filenames from [`readdir()`](<https://man7.org/linux/man-pages/man3/readdir.3.html>), which means they do not contain the slash ("/") character, but subvolume path names do. In addition, the filename returned in the `struct dirent` can only be 255 characters long, which is too restrictive for subvolume names. 

In the end, Poettering thinks that user-space programs do not want to get filenames, they want something that cannot change out from under them. Jeff Layton suggested using [file handles](<https://lwn.net/Articles/375888/>) instead, which Poettering agreed would be better still. Christian Brauner noted that the [`listmount()` system call](<https://lwn.net/Articles/950569/>) uses a new 64-bit mount ID, but there is no way to go from that mount ID to a file descriptor or handle. It would be easy to add, however. 

Overstreet said that he plans to add [firmlinks](<https://one.vg/blog/apple-file-system-firmlink-black-magic>), which is an Apple filesystem (APFS) feature that fits in between hard links and symbolic links. It would use a file handle and filesystem universally unique ID (UUID) to identify a particular file. Amir Goldstein said that [overlayfs](<https://docs.kernel.org/filesystems/overlayfs.html>) also uses those two IDs to identify its files, so Overstreet thought that perhaps that scheme should become a standard for Linux filesystems. 

There are some other missing pieces for file handles, though, he said. There is no system call to go from a file handle to a path. Goldstein said that the ability exists, but it is only reliable for directories. "That's because hard links suck", Overstreet said; Goldstein agreed that was part of it, but Jan Kara said that there are some filesystems that cannot provide that mapping. 

It is getting increasingly difficult to guarantee inode-number uniqueness, Overstreet said. Most of the discussion about his [proposal for a session at LSFMM+BPF](<https://lwn.net/ml/linux-fsdevel/2uvhm6gweyl7iyyp2xpfryvcu2g3padagaeqcbiavjyiis6prl%40yjm725bizncq/>) revolved around the problem and its solutions; it has [come up at earlier summits](<https://lwn.net/Articles/895444/>), as well. The basic problem is that more-recent filesystems (Btrfs, bcachefs) have lots of trouble ensuring that inode numbers are unique across all of the subvolumes/snapshots in a mounted filesystem, which confuses tools like `rsync` and `tar`. 

The 64-bit inode space is simply too small to guarantee uniqueness, he said, but there are various schemes that have been used to make things work. He would rather not be "kicking cans down the road" and thinks filesystem developers need to nudge user-space developers to start using file handles for uniqueness detection "sooner, rather than later". 

#### Inode zero

He noted a recent "kerfuffle" regarding [filesystems that return all inode numbers as zero values](<https://lwn.net/ml/linux-kernel/20240126150209.367ff402%40gandalf.local.home/>), which broke lots of user-space tools. That will become more prevalent over time, so he wondered if it made sense to add a mount option that would intentionally report the same inode number in order to shake out those kinds of problems. Chinner suggested using a sysctl instead, which Overstreet agreed would be a better choice. 

Ted Ts'o said that in order to get user space on board with a switch to using file handles, it is important to make it a cross-OS initiative. Lots of maintainers of user-space tools want to ensure that they work on macOS and the BSDs. If it can get to a point where using file handles is "supported by more forward-leaning, POSIX-like filesystems", the chances will be much better for getting enough of user space converted so that it is possible to return zeroes for inode numbers without breaking everything. It will still be a multi-year effort, which means that it is worth taking the time to try to ensure that it can be adopted more widely than just on Linux. 

Overstreet asked about support for file handles in the other operating systems; Chinner said that anything that supports NFS must have some form of file-handle support. Ts'o agreed but said that the others may not export file-handle information to user space. 

As part of the conversion process, a list of affected programs should be created, Ts'o said. To his "total shock", he found out that the shared-library loader needs unique inode numbers because that is how it distinguishes different libraries, which he found out the hard way. Overstreet wanted to hear that story, but it is a long one that might need a more casual setting to relate, Ts'o said. 

This problem will only get worse in, say, 20 years when 64 bits is even less able to handle the number of inodes, Overstreet said. If the right tools are provided to user-space developers, they will help find and fix all of the problems. But Poettering cautioned that getting rid of the reliance on the uniqueness of inode numbers is going to be extremely difficult. It is used to ensure that the same resources are not loaded multiple times, for example, so it would be better to provide user-space APIs that directly address that problem. 

There was some discussion of various ways to try to add information to the inode number to solve that problem, but there is nothing generalized for all filesystems; it is fair to say there is not any real agreement on how to do that within the filesystem community. Ts'o asked Poettering if file handles, which have more bits to work with, would solve his problems. Poettering said that "file handles are great", but it requires different privileges to query them and they are not implemented on all filesystems, so he still needs a fallback. 

For example, he wondered about getting file handles for procfs files, though it was not entirely clear what the answer to that was. Beyond that, he asked if there was a limit on the size of a file handle; Overstreet said it was a string, so there was no limit. There was some mention of using a hash on the file handles to create a fixed-length quantity, but the end of the session was a bit chaotic, with multiple side discussions all going on at once. 

Brauner got the last word in, pretty much, when he said that he originally had been scared of adding an option to return zero for all inode numbers. But he sees that it makes sense as a tool for educating user space that inode numbers are not unique. There is still a need to provide user space with some kind of API to determine whether two files are actually the same, but that will have to be worked out later—on the mailing list or perhaps at a future summit.
