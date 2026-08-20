---
title: "Snapshots, inodes, and filesystem identifiers"
url: https://lwn.net/Articles/895444/
date: "May 18, 2022"
category: "Filesystems-Btrfs; Network filesystems"
author: "By Jake Edge May 18, 2022 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
May 18, 2022

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2022/>)

A longstanding problem with Btrfs subvolumes and duplicate inode numbers was the topic of a late-breaking filesystem session at the [2022 Linux Storage, Filesystem, Memory-management and BPF Summit](<https://events.linuxfoundation.org/lsfmm/>) (LSFMM). The problem had cropped up in the [bcachefs session](<https://lwn.net/Articles/895266/>) but Josef Bacik deferred that discussion to this just-created session, which he led. The problem is not limited to Btrfs, though, since filesystem snapshots for other filesystems can have similar kinds of problems. 

#### Background

Bacik started with an overview of the problem, in part because he has to re-explain it every few years when it is "discovered" again. Btrfs has subvolumes that contain their own unique inode-number space. Subvolumes can be used for snapshots, so a common use case is to have a subvolume for a home directory so that it can be snapshotted. A snapshot is just a metadata block with a pointer to an existing block and a reference count. That means it has the same files, the same data, and the same inode numbers as the subvolume at the time of the snapshot. 

[ ![\[Josef Bacik\]](https://static.lwn.net/images/2022/lsfmm-bacik-sm.png) ](<https://lwn.net/Articles/895552/>)

That situation confuses tools like `rsync`, so Chris Mason came up with a way to make separate subvolumes appear to be on different filesystems, which meant that the tools would do the right thing. `rsync` (or `find`) will use the `st_dev` value returned by [`stat()`](<https://man7.org/linux/man-pages/man2/lstat.2.html>) to decide if they have traversed into a different filesystem; otherwise, the duplicate inode numbers causes tools to think they have already seen the files. So Btrfs assigns an anonymous block device to each subvolume, which is what it reports via `stat()`. 

That was an easy way to solve the problem, but every time it comes up, "people yell and complain about how terrible and broken it is". There is no other filesystem that does this, he said, so it may not be a great solution, but it did resolve the problem at hand. Internally, Btrfs has a subvolume ID that distinguishes the different inode-number spaces; it is used when Btrfs is being exported via NFS or Ceph to create the unique ID (or filehandle) needed, which works well, he said. 

On the client side, though, the fact that those identical inode numbers are on different subvolumes gets lost, at least for NFS. So if a directory containing a subvolume and its snapshots gets exported, the fact that they are separate subvolumes is not available to the client, so `find` and `rsync` get confused by the duplicate inode numbers. Periodically, someone encounters this problem and "then tells me all the ways that it is easy to fix"; they realize quickly that it is not that easy to fix, Bacik said. The [most recent attempt to do so](<https://lwn.net/Articles/866582/>) was by Neil Brown who tried multiple solutions but it still is not resolved. 

#### Possible solution

What Bacik would like to do is to extend [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>) to report the subvolume ID, which is something that Bruce Fields said would probably work for NFS. The current `st_dev` behavior is still sometimes problematic when tools want to know if two files are on the same filesystem, so he would like `statx()` to report two things: the universally unique ID (UUID) of the containing filesystem and some way to identify the subvolume. Btrfs has a unique object ID for the root of the filesystem, which is a 64-bit value, or the subvolume ID, which is a 128-bit UUID. Either of those could be used by NFS (and others) to determine if the inode numbers are in their own space. But the subvolume UUID is Btrfs-specific, while the root object ID may apply more widely. 

Amir Goldstein asked how the situation was different for ext4 snapshots. Bacik said that the problem was the same for any filesystem that does snapshots. It is only different for snapshots at the block layer, for example using the Logical Volume Manager (LVM). 

On the Zoom chat, Jeff Layton said that Bacik's idea would be formalizing the idea of filesystem and subvolume IDs, which might be a good thing, but other filesystems need to be considered. Bacik agreed, but said that all of the local filesystems he is aware of have a UUID; others wondered about filesystems like FAT. Ted Ts'o said that some FAT filesystems have a 32- or 64-bit ID, but not a UUID. That has come up before in the context of adding a generic mechanism to set the UUID on a filesystem, since some do not have that concept. 

Ts'o also wondered what it meant when Bacik said that a file was in the same filesystem but in a different subvolume. One definition of "the same filesystem" might be that files can be renamed or hard linked within it, but he did not think that was true for Btrfs subvolumes, which Bacik confirmed. Ts'o said it will be important to clearly define what it means for two files to be in the same filesystem, since there may be different expectations among user-space tools. The main use for whether two files are on the same filesystem, Bacik said, is for maintenance tasks to determine which filesystem to mount or unmount, for example. 

#### Not perfect

In general, this mechanism does not have to be perfect, Bacik said, it just needs to give NFS and others some additional information so that they can do whatever it is they need to do. NFS itself works fine, he said, because it uses the unique ID, but `find` and such have problems in those exported directories, so he wants to provide a standard way that network filesystem clients can differentiate those files with the same inode numbers. 

David Howells wondered if `statx()` was the right place for this kind of information; it might make more sense in the [`statfs()`](<https://man7.org/linux/man-pages/man2/statfs.2.html>) information. While Bacik thought that might be a reasonable place to report the UUID for the filesystem, there is still a need to specify which filesystem a given file belongs to, which means `statx()`, he thinks. But, at some level, that is a "nice to have" feature; the real crux of the problem is being able to differentiate the inode-number spaces, which requires a way to identify the subvolume. 

Ts'o pointed out that POSIX-following tools (e.g. `rsync`, `find`) are not going to change to start calling `statx()`; beyond that, those tools are already baked into various enterprise distributions and will need to be supported for a long time. That means the problem will still exist on exported filesystems, unless the NFS client does something different. 

Bacik said that Btrfs has various unique IDs that can be used to recognize and handle the problem, somehow; he just wants to know which IDs are desired and how he should deliver them. Historically, his attitude has been "play stupid games, win stupid prizes"; he suggests not combining the local subvolume and the snapshot in the same export. "Problem solved." 

Bacik said that Christoph Hellwig always suggests that each subvolume have its own VFS mount, but that is a non-starter, because each VFS mount needs its own superblock. That could potentially change, but the problem remains because there are often thousands of subvolumes on a given filesystem. Goldwyn Rodrigues pointed out that each mount gets its own thread, which is "another nightmare to take care of". He said there had been some work on "views" a few years back that had a lightweight superblock for each sub-mount, though he was not sure how far that work progressed. 

Bacik said that he vaguely remembered that work, but, overall, he is tired of talking about this problem. His solution is to extend `statx()` to give NFS and others a way to figure things out. The `st_dev` solution will stay forever, he said, since it works for local filesystems. But for network filesystems, he suggests exporting the UUID of the filesystem and the UUID of the subvolume or the 64-bit object ID of the root, either of which would work. No one present really objected to that plan, so patches should presumably be forthcoming.
