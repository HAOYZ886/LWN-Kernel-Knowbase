---
title: copy_file_range()
url: https://lwn.net/Articles/659523/
date: "October 7, 2015"
category: "System calls-copy file range; reflink"
author: "By Jonathan Corbet October 7, 2015"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 7, 2015

Computers tend to spend a lot of time copying files. Often, the underlying filesystems have ways to accelerate that task, but, as a general rule, copying is still done with a user-space loop reading chunks of data from one file and writing to another. Over the years, there have been efforts to provide access to faster file-copy mechanisms (see [the `reflink()` discussion](<https://lwn.net/Kernel/Index/#reflink>), for example), but no solution has found its way into the mainline. Hope springs eternal, though, so there is a new initiative that might just make it over the bar. 

This initiative is the [`copy_file_range()` patch set](<https://lwn.net/Articles/659197/>) posted by Anna Schumaker. It is built on some earlier work by Zach Brown, with a number of enhancements added on top. This patch set adds a new system call: 

```
ssize_t copy_file_range(int fd_in, loff_t *off_in, int fd_out,
        			    loff_t *off_out, size_t len, unsigned int flags);
```

In its basic form, this system call will copy `len` bytes starting at offset `*off_in` in the file represented by `fd_in` to the offset `*off_out` in `fd_out`. If either of the offsets is NULL, the operation will start at the current file position (the ability to distinguish NULL from zero is the reason why pointers are used for the offsets). The return value is the number of bytes copied, which, as usual, may be less than what was requested. 

In the initial version of `copy_file_range()`, the two files were required to be on the same mount point (but they could not be the same file), and `flags` had to be zero. Anna's work has removed those limitations and added three mutually exclusive flags: 

  * `COPY_FR_COPY` means to copy the data normally, accelerating the work at the filesystem level if possible. 

  * `COPY_FR_REFLINK` asks for the destination file to refer to the existing copy of the data without actually copying it. Some filesystems (Btrfs, for example) are able to share references to file blocks in this way. 

  * `COPY_FR_DEDUP` is like `COPY_FR_REFLINK`, but it only succeeds if the destination range already contains the same data as the source. The end result is files that look the same as before, but which are now sharing the data on-disk. It is thus a way of removing blocks of duplicated data within the filesystem. 

The `COPY_FR_COPY` operation will, in the absence of filesystem-level acceleration, copy the data directly through the kernel page cache; it is essentially a `splice()` operation. Copying through the page cache in this way is clearly more efficient than doing the copy in user space, since it avoids the need to copy the data out of the kernel and back in again. If possible, of course, copying with `COPY_FR_REFLINK` will be the most efficient approach. 

At the system-call level, only one of these flags may be set. But, within the kernel, the code looks like this: 

```
/* Default behavior is to try both. */
        if (flags == 0)
    	flags = COPY_FR_COPY | COPY_FR_REFLINK;
```

So most users are likely to just want to provide zero as the `flags` value; that will result in a "reflink" copy if possible, and an ordinary copy otherwise. 

At the filesystem level, the patch adds a new method to the ever-growing `file_operations` structure: 

```
ssize_t (*copy_file_range)(struct file *in, loff_t off_in,
        			       struct file *out, loff_t off_out,
    			       size_t len, unsigned int flags);
```

The interface is similar to the system call except that `file` structures are used and the offsets are passed directly (reading the offsets from user space is handled at the system-call level). The `flags` value describes how the copy should be performed; as noted above, it might have more than one option set. This function, if provided, should attempt to perform the copy using the method(s) requested. If the function is absent or returns failure, the kernel will, if the `COPY_FR_COPY` flag is set, fall back to copying through the page cache. 

This patch set has been through a number of review cycles at this point, and the flow of comments has slowed considerably. There do not seem to be any concerns that would keep this work out of the 4.4 kernel at this point — though surprises are always possible in kernel development. Barring any such, the long discussion on accelerated copy operations may be just about at an end.
