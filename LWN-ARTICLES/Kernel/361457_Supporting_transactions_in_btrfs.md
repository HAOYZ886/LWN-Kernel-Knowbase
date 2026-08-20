---
title: Supporting transactions in btrfs
url: https://lwn.net/Articles/361457/
date: "November 11, 2009"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet November 11, 2009"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 11, 2009

Much of the fuss involving `fsync()` and crash robustness over the last year was really about how applications can get transactional semantics out of filesystem operations. An application developer often wants to see a set of operations either succeed or fail as a unit, without the possibility of partially-completed operations. Providing ways for applications to get that behavior can be a challenge, though. 

Btrfs has tried to make this capability available to user space by way of the `BTRFS_IOC_TRANS_START` and `BTRFS_IOC_TRANS_END` `ioctl()` calls. There are some real problems with this approach, though. They operate as a pair of system calls, with any operations between the two being treated as a transaction within the filesystem. But if something fails, or if the application never quite gets around to ending the transaction, things will eventually come to a halt. That is why the btrfs code includes this comment: 

There are many ways the trans_start and trans_end ioctls can lead to deadlocks. They should only be used by applications that basically own the machine, and have a very in depth understanding of all the possible deadlocks and enospc problems. 

It is, in other words, a dangerous capability which cannot be made generally available. 

Sage Weil has posted [a patch](<http://lwn.net/Articles/361439/>) taking a rather different approach to the problem. The key idea is to avoid the problem of never-completed transactions by encapsulating the entire thing within a single system call. The result is a new `BTRFS_IOC_USERTRANS` command for `ioctl()`; chances are it will require a bit of work yet, but it could be the base for user-space transactions in the future. 

This command takes a structure which looks something like the following: 

```
struct btrfs_ioctl_usertrans {
    	__u64 num_ops;
    	struct btrfs_ioctl_usertrans_op *ops_ptr;
    	__u64 num_fds;
    	__u64 data_bytes, metadata_ops;
    	__u64 flags;
    	__u64 ops_completed;
        };
```

The `ops_ptr` argument points to an array of `num_ops` individual operations to complete: 

```
struct btrfs_ioctl_usertrans_op {
    	__u64 op;
    	__s64 args[5];
    	__s64 rval;
    	__u64 flags;
        };
```

Here, `op` describes an individual operation. It can be `BTRFS_IOC_UT_OP_OPEN` (open a file), `BTRFS_IOC_UT_OP_CLOSE` (close a file), `BTRFS_IOC_UT_OP_PWRITE` (write to a file), `BTRFS_IOC_UT_OP_UNLINK` (unlink a file), `BTRFS_IOC_UT_OP_LINK` (make a link to a file), `BTRFS_IOC_UT_OP_MKDIR` (create a directory), `BTRFS_IOC_UT_OP_RMDIR` (remove a directory), `BTRFS_IOC_UT_OP_TRUNCATE` (truncate a file), `BTRFS_IOC_UT_OP_SETXATTR` (set an extended attribute), `BTRFS_IOC_UT_OP_REMOVEXATTR` (remove an extended attribute), or `BTRFS_IOC_UT_OP_CLONERANGE` (copy a range of data from one file to another). For each operation, the `args` field contains a set of arguments similar to what one would see for the corresponding system call. One interesting difference is that there are no hard-coded file descriptor numbers; instead, the transaction gets a new file descriptor table and all operations work with indexes into that table. Essentially, transactions work within a file descriptor space separated from that used by the calling process. 

The `flags` field describes how the return value from each operation should be interpreted. It can be contain any of: `BTRFS_IOC_UT_OP_FLAG_FAIL_ON_NE`, `BTRFS_IOC_UT_OP_FLAG_FAIL_ON_EQ`, `BTRFS_IOC_UT_OP_FLAG_FAIL_ON_LT`, `BTRFS_IOC_UT_OP_FLAG_FAIL_ON_GT`, `BTRFS_IOC_UT_OP_FLAG_FAIL_ON_LTE`, and `BTRFS_IOC_UT_OP_FLAG_FAIL_ON_GTE`. In each case, the flag causes the return value to be compared against the passed-in `rval` field; if the comparison is successful, the transaction will fail. 

What happens if the transaction fails? The partially-completed transaction will not be rolled back; btrfs, not being a database, is not really set up to do that. Instead, the number of successfully-completed operations will be passed back to user space. Optionally, the application can provide the `BTRFS_IOC_UT_FLAG_WEDGEONFAIL` flag, causing btrfs to leave the transaction open, locking up the filesystem until the system is rebooted. This may seem like a rather antisocial approach to transaction atomicity, but, if failure is both highly unlikely and highly problematic, that might be the right thing to do. 

A patch like this raises a lot of questions. The first obstacle may be the fact that it requires exporting a number of system call implementations to modules, a change which has been resisted in the past. Kernel code need not normally call functions like `sys_mkdir()`, but this patch does exactly that. Calling system call implementations directly can be a bit tricky on some architectures, and there are good reasons for not making these functions available to modules in general. 

Another problem is that the filesystem has no real way of determining whether a transaction will succeed before jumping into it; the best it can do is reserve some metadata space in accordance with an estimate provided by the application. If transactions are allowed to complete partially, they are no longer transactions. But the alternative of locking up the system can only leave people wondering if there isn't a better way. 

Then, there is a question which was raised on the list: btrfs provides cheap snapshots, why not use them to implement transactions? Using a snapshot would take advantage of existing functionality and should make proper rollback possible. The problem would appear to be performance: btrfs snapshots are not quite _that_ cheap, especially when one considers the need to exclude other activity on the filesystem while the transaction is active. So Chris Mason has [suggested](<https://lwn.net/Articles/361472/>) that the old standby - write-ahead logging - would be preferable because it will perform much better. But, he thinks, the multi-operation `ioctl()` could maybe perform better yet. 

Finally, there would appear to be some real similarities between this API and the rather more general [syslets](<http://lwn.net/Articles/236206/>) mechanism. Syslets have been on the back burner for some time now, but they could come back forward if they seemed like a good way to solve this problem. 

Clearly, like much of btrfs, this new `ioctl()` is a work in progress. If it gets into the mainline, it will be likely to have changed quite a bit on the way. But the problem this patch is trying to solve is real; it's clearly an issue which is worth thinking about.
