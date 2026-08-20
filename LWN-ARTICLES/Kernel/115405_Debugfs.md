---
title: Debugfs
url: https://lwn.net/Articles/115405/
date: "December 13, 2004"
category: Debugging; Filesystems
author: 
---

Kernel hackers often need to be able to export debugging information to user space. This information is not needed for the regular operation of the system, but it can be highly useful for a developer who is trying to figure out why things are behaving strangely. Sometimes putting in a few `printk()` calls is sufficient, but, often, that is not the best way to go. The debugging information may only be useful occasionally, but the printed output clogs up the logs all the time. Using `printk()` also does not help if the developer wishes to be able to change values from user space. 

A common way of making debugging information available only when needed (and possibly for write access) is to create one or more files in a virtual filesystem. There are a few ways in which that can be done: 

  * Creating files in `/proc`. This approach works, but there is little more enthusiasm for creating more files in `/proc` at this point, and the `/proc` filesystem functions can be a bit of a pain to work with. 

  * 2.6 kernels have the `/sys` (sysfs) filesystem. In many cases, debugging information can be put there, but sysfs is meant for information used in administering the system, and the rules for sysfs require that each file contain a single value. For that reason, it is not even possible to use [the seq_file interface](<https://lwn.net/Articles/22355/>) with sysfs. The result is that sysfs is relatively consistent, but it is unwieldy for a developer who wishes to dump out a complicated data structure. 

  * Creating an entirely new filesystem with [libfs](<https://lwn.net/Articles/57369/>). This approach is highly flexible; a developer who creates a new filesystem can write the rules that go with it. The libfs interface makes things relatively simple, but the task of creating a new filesystem is more than most people want to take on just to make some debugging information available - especially since that filesystem will require some debugging of its own. 

As a way of making life easier for developers, Greg Kroah-Hartman has created [debugfs](<https://lwn.net/Articles/115282/>), a virtual filesystem devoted to debugging information. Debugfs is intended to be a relatively easy and lightweight subsystem which gracefully disappears when configured out of the kernel. 

A developer wishing to use debugfs starts by creating a directory within the filesystem: 

```
struct dentry *debugfs_create_dir(const char *name, 
                                          struct dentry *parent);
```

The `parent` argument will usually be `NULL`, causing the directory to be created in the debugfs root. If debugfs is not configured into the system, the return value is `-ENODEV`; a `NULL` return, instead, indicates some other sort of error. 

The general-purpose function for creating a file in debugfs is: 

```
struct dentry *debugfs_create_file(const char *name, mode_t mode,
                                           struct dentry *parent, void *data,
                                           struct file_operations *fops);
```

The structure pointed to by `fops` should, of course, contain pointers to the functions which implement the actual operations on the file. In many cases, most of those functions can be the helpers provided by seq_file, making the task of exporting a file easy. 

Some additional helpers have been provided to make exporting a single value as easy as possible: 

```
struct dentry *debugfs_create_u8(const char *name, mode_t mode, 
                                         struct dentry *parent, u8 *value);
        struct dentry *debugfs_create_u16(const char *name, mode_t mode, 
                                          struct dentry *parent, u16 *value);
        struct dentry *debugfs_create_u32(const char *name, mode_t mode, 
                                          struct dentry *parent, u32 *value);
        struct dentry *debugfs_create_bool(const char *name, mode_t mode, 
                                           struct dentry *parent, u32 *value);
```

Debugfs does not automatically clean up files when a module shuts down, so, for every file or directory created with the above functions, there must be a call to: 

```
void debugfs_remove(struct dentry *dentry);
```

The debugfs interface is quite new, and it may well see changes before finding its way into the mainline kernel. In particular, Greg has [considered](<https://lwn.net/Articles/115418/>) adding a kobject parameter to the creation calls; the kobject would then provide the name for the resulting files.
