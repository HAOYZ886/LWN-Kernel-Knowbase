---
title: A new file_operations method
url: https://lwn.net/Articles/77190/
date: "March 24, 2004"
category: struct file operations
author: 
---

The `file_operations` structure contains pointers to functions which implement I/O operations on files and char devices. These operations include the usual suspects, such as "open", "read", "write", "llseek", etc., along with some more esoteric ones ("sendfile", "get_unmapped_area"). The `file_operations` structure tends not to change very often; changes here can force updating a great many filesystems and drivers. 

The NFS maintainers recently ran into a problem: it is not possible to simultaneously implement the `O_DIRECT` and `O_APPEND` modes over NFS. Rather than silently fail to implement a request to do so, the NFS developers have submitted a patch which adds an `fcntl()` method to the `file_operations` structure. Its prototype is: 

```
long (*fcntl)(unsigned int fd, unsigned int cmd, 
                      unsigned long arg, struct file *filp);
```

The `fd`, `cmd`, and `arg` parameters come straight from user space. A file descriptor is an unusual argument for a `file_operations` method, but the generic `fcntl()` code needs it. `filp` is, as usual, a pointer to the `file` structure for the open file. 

If a module does not provide a `fcntl()` method, the call is handled in the usual way. Otherwise, the new `fcntl()` function should provide a complete implementation of that system call. Typically, the method will perform whatever device- or filesystem-specific work is needed (NFS simply checks for the `O_DIRECT|O_APPEND` combination and returns a failure code if it's there), then pass all four arguments to `generic_file_fcnt()`, which is exported to modules. 

This patch is currently in the -mm tree; it will likely find its way into the mainline sometime after 2.6.5 comes out.
