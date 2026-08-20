---
title: copy_range()
url: https://lwn.net/Articles/550621/
date: "May 15, 2013"
category: System calls; reflink
author: "By Jonathan Corbet May 15, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 15, 2013

Copying a file is a common operation on any system. Some filesystems have the ability to accelerate copy operations considerably; for example, Btrfs can just add another set of copy-on-write references to the file data, and the NFS protocol allows a client to request that a copy be done on the server, avoiding moving the data over the net twice. But, for the most part, copying is still done the old-fashioned way, with the most sophisticated applications possibly using `splice()`. 

There have been various proposals over the years for ways to speed up copy operations ([`reflink()`](<https://lwn.net/Articles/333783/>), for example), but nothing has ever made it into the mainline. The latest attempt is Zach Brown's [`copy_range()` patch](<https://lwn.net/Articles/550604/>). It adds a new system call: 

```
int copy_range(int in_fd, loff_t *in_offset,
    		   int out_fd, loff_t *out_offset, size_t count);
```

The intent of the system call is fairly clear: copy `count` bytes from the input file to the output. It is not said anywhere, but it's implicit in the patch that the two files should be on the same filesystem. 

Inside the kernel, a new `copy_range()` member is added to the `file_operations` structure; each filesystem is meant to implement that operation to provide a fast copy operation. There is no fallback at the VFS layer if `copy_range()` is unavailable, but that looks like the sort of omission that would be fixed before mainline merging. Whether merging will ever happen remains to be seen; this is an area that is littered with abandoned code from previous failed attempts.
