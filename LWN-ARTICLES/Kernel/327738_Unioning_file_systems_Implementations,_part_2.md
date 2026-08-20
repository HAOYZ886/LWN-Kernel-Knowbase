---
title: "Unioning file systems: Implementations, part 2"
url: https://lwn.net/Articles/327738/
date: "April 7, 2009"
category: "Filesystems-Union; Union mounts; Unionfs"
author: "April 7, 2009 This article was contributed by Valerie Aurora"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

April 7, 2009

This article was contributed by Valerie Aurora

In the [first article](<http://lwn.net/Articles/324291/>) in this series about unioning file systems, I reviewed the terminology and major design issues of unioning file systems. In the [second article](<http://lwn.net/Articles/325369/>), I described three implementations of union mounts: Plan 9, BSD, and Linux. In this article, I will examine two unioning file systems for Linux: unionfs and aufs. 

While union mounts and union file systems have the same goals, they are fundamentally different "under the hood." Union mounts are a first class operating systems object, implemented right smack in the middle of the VFS code; they usually require some minor modifications to the underlying file systems. Union file systems, instead, are implemented in the space between the VFS and the underlying file system, with few or no changes outside the union file system code itself. With a union file system, the VFS thinks it's talking a regular file system, and the file system thinks it's talking to the VFS, but in reality both are actually talking to the union file system code. As we'll see, each approach has advantages and disadvantages. 

### Unionfs

Unionfs is the best-known and longest-lived implementation of a unioning file system for Linux. Unionfs development began at SUNY Stony Brook in 2003, as part of the [FiST](<http://www.filesystems.org/>) stackable file system project. Both projects are led by [Erez Zadok](<http://www.cs.sunysb.edu/~ezk/>), a professor at Stony Brook as well as an active contributor to the Linux kernel. Many developers have contributed to unionfs over the years; for a complete list, see the list of past students on the [unionfs](<http://www.filesystems.org/project-unionfs.html>) web page - or read the copyright notices in the unionfs source code. 

Unionfs comes in two major versions, version 1.x and version 2.x. Version 1 was the original implementation, started in 2003. Version 2 is a rewrite intended to fix some of the problems with version 1; it is the version under active development. A design document for version 2 is available at <http://www.filesystems.org/unionfs-odf.txt>. Not all the features described in this document are implemented (at least not in the [publicly available git tree](<http://git.kernel.org/?p=linux/kernel/git/ezk/unionfs.git;a=summary>)); for example, whiteouts are still stored as directory entries with special names, which pollutes the namespace and makes stacking of a unionfs file system over another unionfs file system impossible. 

#### Unionfs basic architecture

The unionfs code is a shim between the VFS and underlying file systems (the branches). Unionfs registers itself as a file system with the VFS and communicates with it using the standard VFS-file system interface. Unionfs supplies various file system operation sets (such as super block operations, which specify how to setup the file system at mount, allocate new inodes, sync out changes to disk, and tear down its data structures on unmount). At the data structure level, unionfs file systems have their own superblock, mount, inode, dentry, and file structures that link to those of the underlying file systems. Each unionfs file system object includes an array of pointers to the related objects from the underlying branches. For example, the unionfs dentry private data (kept in the `d_fsdata` looks like: 

```
/* unionfs dentry data in memory */
        struct unionfs_dentry_info {
                /*
                 * The semaphore is used to lock the dentry as soon as we get into a
                 * unionfs function from the VFS.  Our lock ordering is that children
                 * go before their parents.
                 */
                struct mutex lock;
                int bstart;
                int bend;
                int bopaque;
                int bcount;
                atomic_t generation;
                struct path *lower_paths;
        };
```

The `lower_paths` member is a pointer to an array of path structures (which include a pointer to both the dentry and the `mnt` structure) for each directory with the same path in the lower file systems. For example, if you had three branches, and two of the branches had a directory named "`/foo/bar/`", then, on lookup of that directory, unionfs will allocate (1) a `dentry` structure, (2) a `unionfs_dentry_info` structure with a three-member `lower_paths` array, and (3) two `dentry` structures for the directories. Two members of the `lower_paths` array will be filled with pointers to these dentries and their respective `mnt` structures. The array itself is dynamically allocated, grown, and shrunk according to the number of branches. The number of branches (and therefore the size of the array) is limited by a compile-time constant, `UNIONFS_MAX_BRANCHES`, which defaults to 128 - about 126 more than commonly necessary, and more than enough for every reasonable application of union file systems. The rest of the unionfs data structures - super blocks, dentries, etc. - look very similar to the structure described above. 

The VFS calls the unionfs inode, dentry, etc. routines directly, which then call back into the VFS to perform operations on the corresponding data structures of the lower level branches. Take the example of writing to a file: the VFS calls the `write()` function in the inode's file operations vector. The inode is a unionfs inode, so it calls `unionfs_write()`, which finds the lower-level inode and checks whether it is hosted on a read-only branch. (Unionfs copies up a file on the first write to the data or metadata, not on the first `open()` with write permission.) If the file is hosted on a read-only branch, unionfs finds a writable branch and creates a new file on that branch (and any directories in the path that don't already exist on the selected branch). It then copies up the various associated attributes - file modification and access times, owner, mode, extended attributes, etc. - and the file data itself. Finally, it calls the low-level `write()` file operation from the newly allocated inode and returns the result back to the VFS. 

Unionfs supports multiple writable branches. A file deletion (unlink) operation is propagated through all writable branches, deleting (decrementing the link count) of all files with the same pathname. If unionfs encounters a read-only branch, it creates a whiteout entry in the branch above it. Whiteout entries are named "`.wh.<filename>`", a directory is marked opaque with an entry named "`.wh.__dir_opaque`". 

Unionfs provides some level of cache-coherency by revalidating dentries before operating on them. This works reasonably well as long as all accesses to the underlying file systems goes through the unionfs mount. Direct changes to the underlying file systems are possible, but unionfs cannot correctly handle this in all cases, especially when the directory structure changes. 

Unionfs is under active development. According the [version 2 design document](<http://www.filesystems.org/unionfs-odf.txt>), whiteouts will be moved to small external file system. A inode remapping file in the external file system will allow persistent, stable inode numbers to be returned, making NFS exports of unionfs file systems behave correctly. 

The status of unionfs as a candidate for merging into the mainline Linux kernel is mixed. On the one hand, Andrew Morton merged unionfs into the -mm tree in January 2007, on the theory that unionfs may not be the ideal solution, but it is one solution to a real problem. Merging it into -mm may also prompt developers who don't like the design to work on other unioning designs. However, unionfs has strong NACKs from Al Viro and Christoph Hellwig, among others, and Linus is reluctant to overrule subsystem maintainers. 

The main objections to unionfs include its heavy duplication of data structures such as inodes, the difficulty of propagating operations from one branch to another, a few apparently insoluble race conditions, and the overall code size and complexity. These objections also apply to a greater or lesser degree to other stackable file systems, such as ecryptfs. The consensus at the [2009 Linux file systems workshop](<http://lwn.net/Articles/327601/>) was that stackable file systems are conceptually elegant, but difficult or impossible to implement in a maintainable manner with the current VFS structure. My own experience writing a stacked file system (an in-kernel chunkfs prototype) leads me to agree with these criticisms. 

Stackable file systems may be on the way out. Dustin Kirkland proposed a new design for encrypted file systems that would not be based on stackable file systems. Instead, it would create generic library functions in the VFS to provide features that would also be useful for other file systems. We identified several specific instances where code could be shared between btrfs, NFS, and the proposed ecryptfs design. Clearly, if stackable file systems are no longer a part of Linux, the future of a unioning file system built on stacking is in doubt. 

### aufs

Aufs, short for "Another UnionFS", was initially implemented as a fork of the unionfs code, but was rewritten from scratch in 2006. The lead developer is Junjiro R. Okajima, with some contributions from other developers. The main aufs web site is at [http://aufs.sourceforge.net/](<http://aufs.sourceforge.net/aufs.html>). 

The architecture of aufs is very similar to unionfs. The basic building block is the array of lower-level file system structures hanging off of the top-level aufs object. Whiteouts are named similarly to those in unionfs, but they are hard links to a single whiteout inode in the local directory. (When the maximum link count for the whiteout inode is reached, a new whiteout inode is allocated.) 

Aufs is the most featureful of the unioning file systems. It supports multiple writable branch selection policies. The most useful is probably the "allocate from branch with the most free space" policy. Aufs supports stable, persistent inode numbers via an inode translation table kept on an external file system. Hard links across branches work. In general, if there is more than one way to do it, aufs not only implements them all but also gives you a run-time configuration option to select which way you would like to do it. 

Given the incredible flexibility and feature set of aufs, why isn't it more popular? A quick browse through the source code gives a clue. Aufs consists of about 20,000 lines of dense, unreadable, uncommented code, as opposed to around 10,000 for unionfs and 3,000 for union mounts and 60,000 for all of the VFS. The aufs code is generally something that one does not want to look at. 

The evolution of the aufs source base tends towards increasing complexity; for example, when removing a directory full of whiteouts takes an unreasonably long time, the solution is to create a kernel thread that removes the whiteouts in the background, instead of trying to find a more efficient way to handle whiteouts. Aufs slices, dices, and makes julienne fries, but it does so in ways that are difficult to maintain and which pollute the namespace. More is not better in this case; the general trend is that the fewer the lines of code (and features) in a unioning file system, the better the feedback from other file system developers. 

Junjiro Okajima recently submitted a somewhat stripped down version of aufs for mainline: 

I have another version which dropped many features and the size became about half because such suggestion was posted LKML. But I got no response for it. Additionally I am afraid it is useless in real world since the dropped features are so essential. 

While aufs is used by a number of practical projects (such as the Knoppix Live CD), aufs shows no sign of getting closer to being merged into mainline Linux. 

### The future of unioning file systems development

Disclaimer: I am actively working on union mounts, so my summary will be biased in their favor. 

Union file systems have the advantage of keeping most of the unioning code segregated off into its own corner - modularity is good. But it's hard to implement efficient race-free file system operations without the active cooperation of the VFS. My personal opinion is that union mounts will be the dominant unioning file system solution. Union mounts have always been more popular with the VFS maintainers, and during the VFS session at the recent file systems workshop, Jan Blunck and I were able to satisfactorily answer all of Al Viro's questions about corner cases in union mounts. 

Part of what makes union mounts attractive is that we have focused on specific use cases and dumped the features that have a low reward-to-maintenance-cost ratio. We said "no" to NFS export of unioned file systems and therefore did not have implement stable inode numbers. While NFS export would be nice, it's not a key design requirement for the top use cases, and implementing it would require a stackable file system-style double inode structure, with the attendant complexity of propagating file system operations up and down between the union mount inode and the underlying file system inode. We won't handle online modification of branches other than the topmost writable branch, or modification of file systems that don't go through the union mount code, so we don't have to deal with complex cache-coherency issues. To enforce this policy, Al Viro suggested a per-superblock "no, you REALLY can't ever write to this file system" flag, since currently read/write permissions are on a per-mount basis. 

The `st_dev` and `st_ino` fields in stat will change after a write to a file (technically, an open with write permission), but most programs use this information, along with `ctime/mtime` to decide whether a file has changed - which is exactly what has just happened, so the application should behave as expected. Files from different underlying devices in the same directory may confuse userland programs that expect to be able to rename within a directory - e.g., at least some versions of "`make menuconf`" barf in this situation. However, this problem already exists with bind mounts, which can result in entries with different backing devices in the same directory. Rewriting the few programs that don't handle this correctly is necessary to handle already existing Linux features. 

Changes to the underlying read-only file system must be done offline - when it is not mounted as part of the union. We have at least two schemes for propagating those changes up to the writable branch, which may have marked directories opaque that we now want to see through again. One is to run a userland program over the writable file system to mark everything transparent again. Another is to use the mtime/ctime information on directories to see if the underlying directory has changed since we last copied up its entries. This can be done incrementally at run-time. 

Overall, the solution with the most buy-in from kernel developers is union mounts. If we can solve the `readdir()` problem - and we think we can - then it will be on track for merging in a reasonable time frame.
