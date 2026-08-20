---
title: The end of the fcntl() method
url: https://lwn.net/Articles/98463/
date: "August 18, 2004"
category: fcntl method; struct file operations
author: 
---

Some kernel interfaces last longer than others. The `fcntl()` method is one of the others. It was added to the `file_operations` structure in 2.6.6 with the purpose of giving low-level filesystems and device drivers an opportunity to look at the command being executed from an `fcntl()` system call and, possibly, do something different. The immediate motivation was allowing the NFS code to disallow the combination of the `O_APPEND` and `O_DIRECT` flags, since those two modes cannot work together in that filesystem. Since then, the CIFS filesystem also has made use of it to better handle the `F_NOTIFY` command by getting directory notifications from the remote server. 

In 2.6.8, that operation is gone again. The thinking is that the `file_operations` structure did not really need another general-purpose, multiplexed operation like `fcntl()`. So the method was replaced with two new, carefully-focused methods. The first is: 

```
int (*check_flags)(int flags);
```

This operation, if present, will be called in response to an `fcntl(F_SETFL,...)` system call. It can look at the flags passed in from user space and ensure that they make sense for the device or filesystem in question. 

The other new operation is: 

```
int (*dir_notify)(struct file *filp, unsigned long arg);
```

This is the new method used by CIFS to handle `F_NOTIFY` requests. All other `fcntl()` operations are handled in the core VFS code, as usual. 

[The patch](<https://lwn.net/Articles/98465/>) as merged by Linus fixed the NFS and CIFS code to use the new methods. Unfortunately, nobody tested the NFS changes before the patch was merged, and this change went in just before the final 2.6.8 release came out. The result was an NFS implementation which crashed the kernel, and the need for a quick 2.6.8.1 release.
