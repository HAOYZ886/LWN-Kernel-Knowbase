---
title: securityfs
url: https://lwn.net/Articles/153366/
date: "September 27, 2005"
category: "Security-Security modules"
author: 
---

Some observers might well believe that the kernel has accumulated plenty of special-purpose virtual filesystems. Even so, 2.6.14 will include yet another one: securityfs. This filesystem is meant to be used by security modules, some of which were otherwise creating their own filesystems; it should be mounted on `/sys/kernel/security`. Securityfs thus looks, from user space, like part of sysfs, but it is a distinct entity. 

The API for securityfs is quite simple - it only exports three functions (defined in `<linux/security.h>`). The usual first step will be to create a directory specific to the security module at hand with: 

```
struct dentry *securityfs_create_dir(const char *name, 
                                             struct dentry *parent);
```

If `parent` is `NULL`, the directory will be created in the root of the filesystem. 

That directory can be populated with files using: 

```
struct dentry *securityfs_create_file(const char *name, 
                                              mode_t mode,
    				          struct dentry *parent,
                                              void *data,
    					  struct file_operations *fops);
```

Here, `name` is the name of the file, `mode` is the permissions the file will have, `parent` is the containing directory (or `NULL` for the filesystem root), `data` is a private data pointer, and `fops` is a `file_operations` structure containing the methods which actually implement the file. The calling module must provide operations which make the file behave as desired. Securityfs differs from sysfs in this regard; it makes no attempt to hide the low-level file implementation. As a result, security modules can do ill-advised things like creating highly complex files, providing `ioctl()` operations, and more. Most modules, however, will simply want to provide straightforward `open()`, `read()`, and (maybe) `write()` methods and be done with it. 

All of these files and directories should be cleaned up when the module is unloaded. The same function is used for both files and directories: 

```
void securityfs_remove(struct dentry *dentry);
```

There is no automatic cleanup of files performed, so this step is mandatory. 

Those wanting to see an example of securityfs in action can look at [this patch in 2.6.14](<https://lwn.net/Articles/153370/>) which causes the `seclvl` module to use it.
