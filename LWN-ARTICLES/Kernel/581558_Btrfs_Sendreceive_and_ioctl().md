---
title: "Btrfs: Send/receive and ioctl()"
url: https://lwn.net/Articles/581558/
date: "January 22, 2014"
category: "Btrfs-LWNs guide to; Filesystems-Btrfs"
author: "By Jonathan Corbet January 22, 2014 LWN's guide to Btrfs"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 22, 2014

* * *

[LWN's guide to Btrfs](<https://lwn.net/Articles/576276/>)

At this point, LWN's series on the Btrfs filesystem has covered most of the aspects of working with this next-generation filesystem. This article, the final installment in the series, will deal with a few loose ends that did not fit into the previous articles; in particular, we'll look at the send/receive functionality and a subset of the available `ioctl()` commands that are specific to Btrfs. In both cases, functionality is exposed that is not available in most other Linux filesystems. 

#### Send and receive

The [subvolumes and snapshots article](<https://lwn.net/Articles/579009/>) described a rudimentary scheme for making incremental backups to a Btrfs filesystem. It was simple enough: use `rsync` to make the backup filesystem look like the original, then take a snapshot of the backup to preserve the state of affairs at that particular time. This approach is relatively efficient; `rsync` will only copy changes to the filesystem (passing over unchanged files), and each snapshot will preserve those changes without copying unchanged data on the backup volume. In this way, quite a bit of filesystem history can be kept in a readily accessible form. 

There is another way to do incremental backups, though, if both the original and backup filesystems are Btrfs filesystems. In that case, the Btrfs send/receive mechanism can be used to optimize the process. One starts by taking a snapshot of the original filesystem: 

```
btrfs subvolume snapshot -r _source-subvolume_ _snapshot-name_
```

The _`snapshot-name`_ should probably include a timestamp, since the whole mechanism depends on the existence of regular snapshots at each incremental backup time. The initial snapshot is then copied to the backup drive with a command like: 

```
cd _backup-filesystem_
        btrfs send _path-to-snapshot_ | btrfs receive .
```

This operation, which will copy the entire snapshot, can take quite a while, especially if the source filesystem is large. It is, indeed, significantly slower than just populating the destination filesystem with `rsync` or a pair of `tar` commands. It _might_ work to use one of the latter methods to populate the backup filesystem initially, but using the send/receive chain ensures that things are set up the way those commands expect them to be. 

Note that if the source snapshot is not read-only, `btrfs send` will refuse to work with it. There appears to be no `btrfs` command to set the read-only flag on an existing snapshot that was created as writable, but it is, of course, a simple matter to create a new read-only snapshot of an existing read/write snapshot, should the need arise. 

Once the initial copy of the filesystem is in place, incremental backups can be done by taking a new snapshot on the source filesystem, then running a command like: 

```
cd _backup-filesystem_
        btrfs send -p _path-to-previous-snapshot_ _path-to-new-snapshot_ | btrfs receive .
```

With the `-p` flag, `btrfs send` will only send files (or portions thereof) that have changed since the `_previous-snapshot_` was taken; note that the previous snapshot needs to exist on the backup filesystem as well. Unlike the initial copy, an incremental `send` operation is quite fast — much faster than using a command like `rsync` to find and send changed files. It can thus be used as a low-impact incremental backup mechanism, possibly running many times each day. 

Full use of this feature, naturally, is likely to require some scripting work. For example, it may not be desirable to keep every snapshot on the original filesystem, especially if space is tight there. But it is necessary to keep each snapshot long enough to use it for the next incremental `send` operation; using the starting snapshot would result in the unnecessary copying of a lot of data. Over time, one assumes, reasonably user-friendly tools will be developed to automate these tasks. 

#### Btrfs ioctl() commands

Like most of the relatively complex Linux filesystems, Btrfs supports a number of filesystem-specific `ioctl()` commands. These commands are, as a rule, entirely undocumented; one must go to the (nearly comment-free) source to discover them and understand what they do. This article will not take the place of a proper document, but it will try to point out a few of the more interesting commands. 

Most of the Btrfs-specific commands carry out operations that are available via the `btrfs` command-line tool. Thus, there are commands for the management of subvolumes and snapshots, devices, etc. For the most part, the `btrfs` tool is the best way to access this type of functionality, so those commands will not be covered here. It is amusing to note that several of these commands already come in multiple versions; the first version lacked a field (usually flags to modify the operation) that was added in the second version. 

The structures and constants for all Btrfs `ioctl()` commands should be found in `<linux/btrfs.h>`; some distributions may require the installation of a development package to get that header. 

  * **Cloning files**. The Btrfs copy-on-write (COW) mechanism can be used to make copies of files that share the underlying storage, but which still behave like separate files. A file that has been "cloned" in this way behaves like a hard link as long as neither the original file nor the copy is modified; once a change is made, the COW mechanism copies the modified blocks, causing the two files to diverge. Cloning an entire file is a simple matter of calling: 

```
status = ioctl(dest, BTRFS_IOC_CLONE, src);
```

Where `dest` and `src` are open file descriptors indicating the two files to operate on; `dest` must be opened for write access. Both files must be in the same Btrfs filesystem. 

To clone a portion of a file's contents, one starts with one of these structures: 

```
struct btrfs_ioctl_clone_range_args {
          	__s64 src_fd;
          	__u64 src_offset, src_length;
          	__u64 dest_offset;
            };
```

The structure is then passed as the argument to the `BTRFS_IOC_CLONE_RANGE` `ioctl()` command: 

```
status = ioctl(dest, BTRFS_IOC_CLONE_RANGE, &args);
```

As with `BTRFS_IOC_CLONE`, the destination file descriptor is passed as the first parameter to `ioctl()`. 

Note that the clone functionality is also available in reasonably modern Linux systems using the `--reflink` option to the `cp` command. 

  * **Explicit flushing**. As with any other filesystem, Btrfs will flush dirty data to permanent storage in response to the `fsync()` or `fdatasync()` system calls. It is also possible to start a synchronization operation explicitly with: 

```
u64 transid;
        
            status = ioctl(fd, BTRFS_IOC_START_SYNC, &transid);
```

This call will start flushing data on the filesystem containing `fd`, but will not wait for that operation to complete. The optional `transid` argument will be set to an internal transaction ID corresponding to the requested flush operation. Should the need arise to wait until the flush is complete, that can be done with: 

```
status = ioctl(fd, BTRFS_IOC_WAIT_SYNC, &transid);
```

The `transid` should be the value returned from the `BTRFS_IOC_START_SYNC` call. If `transid` is a null pointer, the call will block until the current transaction, whatever it is, completes. 

  * **Transaction control**. The flush operations can be used by an application that wants to ensure that one transaction completes before starting something new. Programmers who want to live dangerously, though, can use the `BTRFS_IOC_TRANS_START` and `BTRFS_IOC_TRANS_END` commands (which take no arguments) to explicitly begin and end transactions within the filesystem. All filesystem operations made between the two calls will become visible to other processes in an atomic manner; partially completed transactions will not be seen. 

The transaction feature seems useful, but one should heed well this comment from `fs/btrfs/ioctl.c`: 

```
/*
             * there are many ways the trans_start and trans_end ioctls can lead
             * to deadlocks.  They should only be used by applications that
             * basically own the machine, and have a very in depth understanding
             * of all the possible deadlocks and enospc problems.
             */
```

Most application developers, one might imagine, lack this "very in depth understanding" of how things can go wrong within Btrfs. Additionally, there seems to be no way to abort a transaction; so, for example, if an application crashes in the middle of a transaction, the transaction will be ended by the kernel and the work done up to the crash will become visible in the filesystem. So, for most developers considering using this functionality, the right answer at this point is almost certainly "don't do that." Anybody who wants to try anyway will need the `CAP_SYS_ADMIN` capability to do so. 

There are quite a few more `ioctl()` commands supported by Btrfs, but, as mentioned above, most of them are probably more conveniently accessed by way of the `btrfs` tool. For the curious, the available commands can be found at the bottom of `fs/btrfs/ioctl.c` in the kernel source tree. 

#### Series conclusion

At this point, LWN's series on the Btrfs filesystem concludes. The major functionality offered by this filesystem, including device management, subvolumes, snapshots, send/receive and more, has been covered in the five articles that make up this set. While several developers have ideas for other interesting features to add to the filesystem, chances are that most of that feature work will not go into the mainline kernel anytime soon; the focus, at this point, is on the creation of a stable and high-performance filesystem. 

There are few knowledgeable developers who would claim that Btrfs is fully ready for production work at this time, so that stabilization and performance work is likely to go on for a while. That said, increasing numbers of users are putting Btrfs to work on at least a trial basis, and things are getting more solid. Predictions of this type are always hard to make successfully, but it seems that, within a year or two, Btrfs will be accepted as a production-quality filesystem for an increasingly wide range of use cases.
