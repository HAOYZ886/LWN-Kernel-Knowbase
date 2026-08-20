---
title: Unifying filesystems with union mounts
url: https://lwn.net/Articles/312641/
date: "December 24, 2008"
category: "Filesystems-Union; Union mounts"
author: "December 24, 2008 This article was contributed by Goldwyn Rodrigues"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

December 24, 2008

This article was contributed by Goldwyn Rodrigues

Unification of filesystems is the concept of mounting several filesystems on a single mount point, with the resulting mount showing the logical combination of all the filesystems. Traditionally, when a filesystem is mounted on a directory, the existing contents of the directory are masked, and the content of the latest mounted filesystem is shown. These masked files are available only after the mounted filesystem is unmounted. Even though these files exist, they are inaccessible to the user. Union mount overcomes this by providing access to all directories and files present in the directory, even after a mount. 

In the kernel, the filesystems are stacked in order of their mount sequence, the first mounted filesystem is at the bottom of the mount stack, and the latest mount is at the top of the stack. Only the files and directories of the top of the mount stack are visible. With union mounts, directory entries from the lower filesystems are merged with the directory entries of upper filesystem, thus making a logical combination of all mounted filesystems. Files with the same name in a lower filesystem are masked, as the upper one takes precedence. 

Union mounts could be used to update packages of a distribution on a DVD. A writable filesystem could be mounted over the read-only filesystem on the DVD. All new and updated package files would be written to the writable, topmost filesystem, while hiding the duplicate files of the read-only media, or even deleting files (this is done through white-outs discussed later). This allows the user to change any of the files on the system, with the new file stored transparently in the image. Such a setup could be used to roll-up an updated DVD, or maintain a package repository with the latest packages for network installs. 

As compared to other implementations, such as [unionFS](<http://lwn.net/Articles/217084/>), union mounts try to do all directory entry unification handling in the VFS layer, instead of creating a new filesystem type. Some of the advantages of this approach are: 

  * Simple and Lightweight Design: Since all merges happen inside VFS, there is no need for an additional filesystem layer to maintain and merge metadata. 
  * No need to re-iterate the mount stack by the user while mounting: the user is not required to list the directories participating in the union as a part of the mount command. Only the mount point is enough. 
  * Bind mount works without any problems: this is a VFS feature to remount part of the filesystem hierarchy at additional mount points. 

[Union mount](<http://lwn.net/Articles/308920/>), developed by Jan Blunck, Bharta B Rao, and Miklos Szeredi, is the first step in unifying mounts in the VFS. The patch implementation is similar to that of the [Plan 9/Inferno operating system](<http://plan9.bell-labs.com/plan9dist/>). Currently, it only does namespace unification at the root directory level and not in the subdirectories. 

To mount directories through union mount, the mount command must be modified to recognize and set the union mount options. The util-linux patches that update the mount command can be found at [ ftp://ftp.suse.com/pub/people/jblunck/union-mount/](<ftp://ftp.suse.com/pub/people/jblunck/union-mount/>)

As an example, consider the following directory structure of two filesystems: 

> ![\[sdb filesystem\]](https://static.lwn.net/images/union/sdb.png) |  ![\[sdc filesystem\]](https://static.lwn.net/images/union/sdc.png)  
> ---|---  
  
Issuing the following commands will perform a union mount: 

```
# mount /dev/sdb /mnt
        # ls /mnt
        dir1 file1 link1
    
        # mount --union /dev/sdc /mnt
        # ls /mnt
        dir1 dir4 file1 link1
```

After the union, the directory structure looks like: 

> ![\[after union mount\]](https://static.lwn.net/images/union/union.png)

Unmounting the /mnt directory unwinds the filesystem mount stack: 

```
# umount /mnt
        # ls /mnt
        dir1 file1 link1
```

The filesystems are stacked in the mount order in the kernel. The `MNT_UNION` flag in `vfsmnt` is set while mounting union mounts. This helps to identify that the directory entries of the stacked filesystems are supposed to be merged. While performing the lookup sequence, if the `MNT_UNION` flag is set, all root directory entries of all filesystems are scanned. Scanning happens from top of the filesystem stack to bottom, and the first matching entry is returned. This way any duplicate entries in underlying filesystems are automatically ignored. 

Similarly, for the `readdir()` call, the directory entries are read from the topmost union mount directory to the lowest, and collected in the cache. The cache is responsible for collecting and keeping the directory entries across the stacked filesystem, with different callbacks for each filesystem. Like regular files, directories are seekable and the position of the following read is marked by the file position `filp->f_pos`. When reading from directories across filesystems, it is possible that the file position exceeds the inode size of the directory where it is merged. In such a situation, the file position is rearranged to select the correct directory in the union stack. This is done by subtracting the inode size if the file position exceeds it and selecting the next member of the union. 

This works for filesystems such as ext2 that use flat file directories. The directory entry offsets are arranged linearly and are always smaller than the inode size of the directory. However, some filesystems return special cookies as directory entry offsets which are unrelated to the position in the directory or the inode size. Updating `file->f_pos` to accommodate more directories does not not work for such filesystems. 

There can be multiple calls to `readdir()`/`getdents()` routines for reading the entries of a single directory. Currently, the union directory cache is not maintained across these calls. Instead, for every call the previously read entries are re-read into the cache and newly read entries are compared against these for duplicates before being returned to user space. The developers are working on making this efficient by maintaining the cache across `readdir()`/`getdents()` calls. 

#### Future Plans: Writable Unions

Currently, the namespace unification is limited to the root filesystem directory entries. Future plans, known as writable unions, would come close to the implementations of unionfs namespace unification. Directory entry merging would not be limited to the root filesystem, but would be done for subdirectories as well. Though these patches have been developed, they still require some time and clean up for the mainline. 

Using the example above, a writable union mount of the two filesystems would contain: 

> ![\[after writeable union\]](https://static.lwn.net/images/union/w_union.png)

Note that dir1 directory now contains both file_b1 and file_c1. 

All writes are directed to the topmost mounted filesystem if it is mounted read-write. Mounting a new filesystem upon the current union mount makes all filesystems lower in the stack read-only, though the unified namespace would appear read-write to the user. Any modifications in the files of lower filesystems is handled through copy-on-write. If a file belonging to the lower layers of the stack is opened, the entire file is copied on the topmost filesystem on the stack. This is also known as copy-up, where the file is copied to the topmost layer if it has to record a change. While performing a copy-up, the directory path of the file is also recreated on the topmost filesystem, so that the next time it is mounted as a union, it appears in the same location. The older file gets masked during the directory merge the next time the filesystems are union-mounted in the same order. 

Rename on union mounts is handled through `-EXDEV`. `-EXDEV` is returned in a `rename()` operation if the source and destination file paths are on different mounted filesystems. In such a case, the application, such as `mv`, resorts to a copy operation, and unlinks the file from which the filesystem moved. On union mounts, since any writes are performed in the topmost layer, a move operation to directories in the lower layers returns `-EXDEV`, which means the application must copy the file to the new directory. If both the source and destination of the `rename()` operation are in the topmost later, the traditional rename method is used. 

Deletion of files is handled by a special file type called white-outs. The white-out file type is similar to negative dentries: they describe a filename which isn't there. This is used to mark a file in the lower read-only filesystem as deleted, since only the topmost layer can be modified. However, white-outs would require support from all the filesystems, to store and recognize such a special file type. Currently, there is a special type, `DT_WHT` defined in `include/linux/fs.h` which defines a white-out, but is not in use. 

Directory namespace unification is a tough task. FreeBSD implementations gave up after calling it "messy code", while unionfs entered the -mm tree for a brief period, it did not make it to mainline. Since the unification is a pathname-based it is best handled in the VFS instead of using a separate stacked filesystem. The union mount offers a cleaner and more lightweight approach for merging directories, however getting it to adhere to POSIX compliant directory calls such as `telldir()` or `seekdir()` is still a challenge and is currently being worked on. 

The git repository to track union mounts is located at: 

```
git://git.kernel.org/pub/scm/linux/kernel/git/mszeredi/vfs.git
```

under the `union-dir` branch. The union mounts developers intend to release the patches in a phased manner, starting with the current patch of root directory level merging. Further developments would see patches related to merging at the subdirectory level as well.
