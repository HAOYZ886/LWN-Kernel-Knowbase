---
title: Cleaning up the block driver API
url: https://lwn.net/Articles/247072/
date: "August 28, 2007"
category: "Block layer-Block drivers; Device drivers-Block drivers"
author: "By Jonathan Corbet August 28, 2007"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
August 28, 2007

Once upon a time, block device drivers implemented the same `file_operations` structure used by char drivers - despite the fact that block drivers are quite different and many of the `file_operations` methods had no relevance to them. By the 2.4 release, though, the block driver API had been significantly reworked, and `struct file_operations` was no longer used. Instead, block drivers have a `block_device_operations` structure containing many of the driver's exported operations. "Many" because certain other operations, including the ones which actually enqueue I/O requests, end up being stored in the request queue structure instead. 

When the move to `block_device_operations` was done, a number of methods were carried over directly from the `file_operations` vector with their prototypes unchanged. Doing things this way minimized the pain for driver maintainers, but it led to some interesting interface artifacts. For example, consider the `open()` method: 

```
int (*open)(struct inode *ino, struct file *filp);
```

When a char device or an actual file is being opened, `filp` points to the internal `file` structure used by the kernel to manage the open file. If a user-space process opens a block device directly, `filp` will be used in the same way. Most of the time, though, block devices are opened by the kernel as a step toward mounting a filesystem stored there. In that case, there is no associated `file` structure. That's why a perusal of the source reveals code like this: 

```
/*
         * This crockload is due to bad choice of ->open() type.
         * It will go away.
         * For now, block device ->open() routine must _not_
         * examine anything in 'inode' argument except ->i_rdev.
         */
        struct file fake_file = {};
        struct dentry fake_dentry = {};
        fake_file.f_mode = mode;
        fake_file.f_flags = flags;
        fake_file.f_path.dentry = &fake_dentry;
        fake_dentry.d_inode = bdev->bd_inode;
```

Al Viro (who is responsible for much of the current API) has [taken a look at this problem](<http://lwn.net/Articles/246919/>) and others. In the case of `open()`, there is very little of the information passed in the `inode` and `file` structure pointers which is actually used by drivers. And some of that is used in hazardous ways - any driver which depends on anything in `fake_file` lasting beyond the `open()` call will find itself in trouble. There are other issues with the API as well, leading Al to propose some significant changes. The result, which is almost certain to be merged when it is ready (possibly as soon as 2.6.24), will be a cleaner block driver API - at the cost of changes for every existing driver. 

The first change will be to move some of the flags found in `f_flags` over to `f_mode`, which is not subject to being changed by `fcntl()` calls from user space. As part of the move, drivers will be expected not to change those flags - or any other part of the `file` structure. This change will enable a cleanup of some code in the much-maligned floppy driver, which currently stores some information in that structure at `open()` time. 

The new `open()` prototype is projected to be: 

```
int (*open)(struct block_device *bdev, mode_t mode);
```

Where `mode` has the usual read/write flags, but also some of the other `open()`-time flags like `O_NDELAY`. This value will not be changed by the drivers and will not necessarily exist in any sort of `file` structure. It will be stored safely in an undisclosed location by the kernel and will be available at `release()` time, when some drivers will need access to those flags. 

Speaking of `release()`, that function, too, currently has an old prototype: 

```
int (*release)(struct inode *ino, struct file *filp);
```

In this case, `filp` is often passed as `NULL` by the kernel, forcing drivers to check the value and implement some sort of default behavior in the lack of a `file` structure. But, sometimes, drivers need to know about some of the flags which were provided at `open()` time. So the new `release()` method will look something like: 

```
int (*release)(struct gendisk *disk, mode_t mode);
```

The changes do not stop there. Al points out that there is a bit of confusion in the `ioctl()` interface: 

```
int (*ioctl)(struct inode *ino, struct file *filp, unsigned cmd, 
                     unsigned long arg);
        long (*unlocked_ioctl)(struct file *filp, unsigned cmd, unsigned long arg);
        long (*compat_ioctl) (struct file *filp, unsigned cmd, unsigned long arg);
```

The different versions have different arguments - and even different return types. Once again, drivers tend not to care about most of what can be found in the `inode` and `file` structures - even when those structures exist. So the new form of the `ioctl()` methods will be: 

```
int (*ioctl)(struct block_device *bdev, mode_t mode, unsigned int cmd, 
                     unsigned long arg);
        int (*compat_ioctl)(struct block_device *bdev, mode_t mode, unsigned int cmd,
                            unsigned long arg);
```

Note that `unlocked_ioctl()` is gone: it is arguably past time to get rid of the big kernel lock (BKL) in the block `ioctl()` implementation. So any driver still using the locked version (`ioctl()` in the old API) will be modified to take the BKL internally. Any block driver which still requires the BKL is probably in need of a more serious review, though. 

As of this writing, there have been no arguments against the change. [The word from Linus](<https://lwn.net/Articles/247100/>) is: 

From your description, I have no objections - everything sounds good. My only concern is how painful the patch ends up being (and a worry about whether this will affect a metric truck-load of external modules? That said, I can't really see us worrying about those) 

Al claims to have a patch in progress and ready to be posted soon, and that the amount of pain should be relatively small - for in-tree drivers, anyway. For those maintaining out-of-tree block drivers, the writing is on the wall: a significant API change is coming.
