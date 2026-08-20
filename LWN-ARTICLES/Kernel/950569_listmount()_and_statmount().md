---
title: listmount() and statmount()
url: https://lwn.net/Articles/950569/
date: "November 10, 2023"
category: "Filesystems-Mounting; Releases-6.8; System calls"
author: "By Jonathan Corbet November 10, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
November 10, 2023

Years ago, the list of mounted filesystems on a Unix or Linux machine was relatively short and static. Adding a filesystem, which typically involved buying a new drive, happened rarely. In contrast, contemporary systems with a large number of containers can have a long and dynamic list of mounted filesystems. As was [discussed](<https://lwn.net/Articles/934469/>) at the [2023 LSFMM+BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023/>), the Linux kernel's mechanism for providing information about mounted filesystems has not kept up with this change, leading to system-management headaches. Now, two new system calls [proposed](<https://lwn.net/ml/linux-kernel/20231025140205.3586473-1-mszeredi@redhat.com/>) by Miklos Szeredi look set to provide some much-needed pain relief. 

Even in the absence of containers, the list of mounted filesystems on a typical Linux system has grown, partly as a result of an increase in the number of virtual filesystems provided by the kernel. For example, on your editor's basic desktop system, `/proc/self/mountinfo` lists 34 mounts, few of which correspond to partitions on actual storage devices. As this virtual file gets longer, it becomes harder for system-management tools (and humans too) to work with. The new system calls, called `listmount()` and `statmount()`, provide an alternative to digging through the `mountinfo` file. 

Before implementing those system calls, though, Szeredi had to address a related problem. Every mount in the system is assigned a mount ID to identify it; that ID, which is available from [`statx()`](<https://man7.org/linux/man-pages/man2/statx.2.html>), is the obvious way to talk about mounts in a new system call. If, however, a filesystem is unmounted, its ID will be reused by the kernel to identify a new mount in the future, making it into an ambiguous identifier. The obvious solution is to stop reusing mount IDs, since nothing really requires that behavior. 

Unfortunately, there are user-space programs that assume that the mount ID is a 32-bit quantity, despite the fact that it is defined as `_u64` in the `statx()` system call. Systemd [was identified](<https://lwn.net/Articles/932648/>) as one of those programs during the LSFMM+BPF discussions. Making a 32-bit mount ID unique over the life of the system only allows for 4 billion mounts, which is apparently constraining in some settings; it also could possibly be deliberately overflowed by an attacker. So a new, even more explicitly 64-bit, mount ID is needed. The patch series adds it, along with a new `statx()` flag (`STATX_MNT_ID_UNIQUE`) that causes the unique ID to be returned rather than the 32-bit ID (which, of course, cannot go away). As a way of avoiding confusion between the two IDs, the lowest unique mount ID is set to 232. 

With that in place, the two system calls can be added; the first is: 

```
struct __mount_arg {
    	__u64 mnt_id;
    	__u64 request_mask;
        };
    
        int listmount(const struct __mount_arg *req, u64 *buf, size_t bufsize,
        		  unsigned int flags);
```

A call to `listmount()` will return a list of filesystems mounted below the mount point identified by `req->mnt_id`, where that ID must be of the unique variety. The results are returned as an array of (unique) mount IDs in `buf`, which is `bufsize` in length. Normally, unreachable mounts (that may, for example, be mounted in a different mount namespace) are omitted; adding `LISTMOUNT_UNREACHABLE` to `flags` will cause those to be listed as well; this option requires the `CAP_SYS_ADMIN` capability. The `LISTMOUNT_RECURSIVE` flag will cause `listmount()` to do a depth-first traversal of the hierarchy below the starting mount point and list all mounts found there; otherwise, only direct child mounts are returned. The return value is the number of mount IDs returned (or an error code). 

The `request_mask` field of the `req` structure is not used by `listmount()` and must be zero. 

The other call, `statmount()`, returns the details of a given mount: 

```
int statmount(const struct __mount_arg *req, struct statmnt *buf,
        		  size_t bufsize, unsigned int flags);
```

For this call, `req->mnt_id` identifies the mount of interest as before, while `req->request_mask` tells the kernel which information is requested. The `flags` value must be zero, and `buf` points to a buffer (of `bufsize` bytes) that begins with this structure: 

```
struct statmnt {
    	__u32 size;		/* Total size, including strings */
    	__u32 __spare1;
    	__u64 mask;		/* What results were written */
    	__u32 sb_dev_major;	/* Device ID */
    	__u32 sb_dev_minor;
    	__u64 sb_magic;		/* ..._SUPER_MAGIC */
    	__u32 sb_flags;		/* MS_{RDONLY,SYNCHRONOUS,DIRSYNC,LAZYTIME} */
    	__u32 fs_type;		/* [str] Filesystem type */
    	__u64 mnt_id;		/* Unique ID of mount */
    	__u64 mnt_parent_id;	/* Unique ID of parent (for root == mnt_id) */
    	__u32 mnt_id_old;	/* Reused IDs used in proc/.../mountinfo */
    	__u32 mnt_parent_id_old;
    	__u64 mnt_attr;		/* MOUNT_ATTR_... */
    	__u64 mnt_propagation;	/* MS_{SHARED,SLAVE,PRIVATE,UNBINDABLE} */
    	__u64 mnt_peer_group;	/* ID of shared peer group */
    	__u64 mnt_master;	/* Mount receives propagation from this ID */
    	__u64 propagate_from;	/* Propagation from in current namespace */
    	__u32 mnt_root;		/* [str] Root of mount relative to root of fs */
    	__u32 mnt_point;	/* [str] Mountpoint relative to current root */
    	__u64 __spare2[50];
    	char str[];		/* Variable size part containing strings */
        };
```

The kernel will not necessarily fill in all of the fields of this structure; instead, it provides the information indicated in the `req->request_mask` field. The available requests are: 

  * `STMT_SB_BASIC`: "basic" superblock data from the mount, specifically the `sb_dev_major`, `sb_dev_minor`, `sb_magic`, and `sb_flags` fields. 
  * `STMT_MNT_BASIC`: more basic data: `mnt_id`, `mnt_parent_id`, `mnt_id_old`, `mnt_parent_id_old`, `mnt_attr`, `mnt_propagation`, `mnt_peer_group`, and `mnt_master`. 
  * `STMT_PROPAGATE_FROM`: fills in the `propagate_from` field. (See [the shared subtrees documentation](<https://www.kernel.org/doc/html/latest/filesystems/sharedsubtree.html>) for details on mount propagation). 

Requests that yield strings are handled a bit differently. The actual string data will be written in the memory after the structure (`buf` must be big enough to hold that data), and the offset of the beginning of the string will be stored in the relevant structure field. The string-returning requests are: 

  * `STMT_FS_TYPE`: stores the string representation of the filesystem type after the structure, placing the offset of the string in `fs_type`. 
  * `STMT_MNT_ROOT`: stores the path to root filesystem, with the offset in `mnt_root`. 
  * `STMT_MNT_POINT`: stores the path to the mount point, with the offset in `mnt_point`. 

On a successful return, the `mask` field will be set to indicate which of the other fields in the structure were written by the kernel. 

VFS maintainer Christian Brauner has [accepted this series](<https://lwn.net/ml/linux-kernel/20231101-urenkel-banal-b232d7a3cbe8@brauner/>), with the probable objective of merging it for 6.8. He made a few changes in the process, though: `struct statmnt` was renamed to `struct statmount` and `struct __mount_arg` became `struct mnt_id_req`. ""Libraries can expose this in whatever form they want but we'll also have direct consumers. I'd rather have this struct be underscore free and officially sanctioned."" The result has not yet shown up in linux-next, but seems likely to do so once the 6.7 merge window has closed. 

It would not be surprising if the interface provided by C libraries differed from that shown here. The `mnt_id_req` structure, for example, is used to [simplify compatibility](<https://lwn.net/ml/linux-kernel/44631c05-6b8a-42dc-b37e-df6776baa5d4@app.fastmail.com/>) across multiple architectures, but user-space libraries do not have the same concerns and may not wish to expose that structure. Details like that are unlikely to be worked out before these system calls show up in a released kernel. Eventually, though, there will be a better and easier way to obtain information about which filesystems are mounted.
