---
title: "The Btrfs inode-number epic (part 1: the problem)"
url: https://lwn.net/Articles/866582/
date: "August 20, 2021"
category: "Filesystems-Btrfs"
author: "By Jonathan Corbet August 20, 2021"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
August 20, 2021

Unix-like systems — and their users — tend to expect all filesystems to behave in the same way. But those users are also often interested in fancy new filesystems offering features that were never envisioned by the developers of the Unix filesystem model; that has led to a number of interesting incompatibilities over time. Btrfs is certainly one of those filesystems; it provides a long list of features that are found in few other systems, and some of those features interact poorly with the traditional view of how filesystems work. Recently, Neil Brown has been trying to resolve a specific source of confusion relating to how Btrfs handles inode numbers. 

One of the key Btrfs features is [subvolumes](<https://lwn.net/Articles/579009/>), which are essentially independent filesystems maintained within a single storage volume. Snapshots are one commonly used form of subvolume; they allow the storage of copies of the state of another subvolume at a given point in time, with the underlying data shared to the extent that it has not been changed since each snapshot was taken. There are other applications for subvolumes as well, and they tend to be heavily used; Btrfs filesystems can contain thousands of subvolumes. 

Btrfs subvolumes bring some interesting quirks with them. They can be mounted independently, as if they were separate filesystems, but they also appear as a part of the filesystem hierarchy as seen from the root. So one _can_ mount subvolumes, but a subvolume can be accessed without being mounted if a higher-level directory _is_ mounted. Imagine, for example, that `/dev/sda1` contains a Btrfs filesystem that has been mounted on `/butter`. One could create a pair of subvolumes with commands like: 

```
# cd /butter
        # btrfs subvolume create subv1
        # btrfs subvolume create subv2
```

The root of `/butter` will now appear to contain two directories (`subv1` and `subv2`): 

```
# tree /butter
        /butter
        ├── subv1
        └── subv2
    
        2 directories, 0 files
```

They behave like directories most of the time but, since they are actually subvolumes, there are some differences; one cannot rename a file from one to the other, for example. A suitably privileged user can now mount either `subv1` or `subv2` (or both) as independent filesystems. But, as long as `/butter` remains mounted, _both_ subvolumes are visible as if they were part of the same filesystem. There are some interesting consequences from this behavior, as will be seen. 

Btrfs uses a subvolume ID number internally to identify subvolumes, but there is no way to make that number directly visible to user space. Instead, the filesystem allocates a separate device number (the usual major/minor pair) for each subvolume; that number can be seen with a system call like [`stat()`](<https://man7.org/linux/man-pages/man2/lstat.2.html>). If the subvolumes are not explicitly mounted, though, numbers do not show up in files like `/proc/self/mountinfo`, leading to inconsistent views of how the filesystem is put together. [**Update** : as Brown pointed out to us privately, the numbers do not show up there even if the subvolumes _are_ explicitly mounted.] A call to `stat()` on a file within a subvolume will return a device number that does not exist in files like `mountinfo`, a situation that occasionally confuses unaware applications. 

It gets worse. Since Btrfs has a unique internal ID for each subvolume, it feels no particular need to keep inode numbers unique across those subvolumes. As a result, a process walking a Btrfs filesystem from the root may well encounter multiple files with the same inode number. Tools like `find` use inode numbers as a way of tracking which files they have already seen and detecting filesystem loops. For a locally mounted Btrfs filesystem, things mostly work as expected because, even though two files on different subvolumes may have the same inode number, they will have different device numbers and are thus distinct. 

The kernel's NFS daemon, though, has a harder time of things. It cannot present all of those artificial device numbers to NFS clients, because that would require all of the subvolumes — again, possibly thousands of them — to show up as separate mounts on the client. So a Btrfs filesystem exported via NFS shows the same device number (the device number of the root) on all subvolumes. That works most of the time, but it can make it impossible to use a tool like `find` on an NFS-mounted Btrfs filesystem with subvolumes. The single device number makes it impossible to distinguish files with the same inode number on different subvolumes, causing `find` to abort with a message about filesystem loops. This leads to occasional [complaints](<https://lwn.net/ml/linux-btrfs/20210310074620.GA2158@tik.uni-stuttgart.de/>) from users and a desire to somehow improve the situation. 

These problems are not new; they have been known and understood for years. The level of complaints seems to be rising, though, perhaps as a consequence of increased use of Btrfs in production situations. In theory, the way to solve these problems is understood as well — though not all developers have the same understanding, as Neil Brown found out when he took on the task of fixing Btrfs filesystems exported via NFS. The [second and last article in this series](<https://lwn.net/Articles/866709/>), published on August 23, explores various attempted solutions to this problem and why it turns out to be so hard to fix.
