---
title: "tee() with your splice()?"
url: https://lwn.net/Articles/179492/
date: "April 11, 2006"
category: Pipes; splice; tee
author: 
---

The [new `splice()` system call](<http://lwn.net/Articles/178199/>) was covered here last week. As was predicted then, this new kernel API has continued to evolve; many of the non-fix patches going into the post-2.6.17-rc1 mainline involved changes to `splice()`. 

For starters, the prototype of the `splice()` system call has changed: 

```
long splice(int fd_in, loff_t *off_in, int fd_out, loff_t *off_out,
                    size_t len, unsigned int flags);
```

The two offset values (`off_in` and `off_out`) are new; they indicate where each file descriptor should be positioned prior to beginning the transfer of data. Note that these offsets are passed via pointers; user space can use a `NULL` pointer to indicate that the current offset should be used. Note also that these offsets do not work like the offsets in `pread()` or `pwrite()`: they will actually change the offset associated with the file descriptor. Providing an offset for a file descriptor associated with a pipe is an error. 

Internally, the `splice()` code has seen a couple of interesting changes. One of them (in the regular pipe code, actually) is the creation of a new `pipe_inode_info` structure to represent the core machinery behind a pipe. This structure can stand apart from the normal `inode` structure. Many of the internal interfaces have been changed to use the new structure, including the new methods in the `file_operations` structure: 

```
ssize_t (*splice_write)(struct pipe_inode_info *pipe, 
                                struct file *out, size_t len, 
    			    unsigned int flags);
        ssize_t (*splice_read)(struct file *in, struct pipe_inode_info *pipe, 
                               size_t len, unsigned int flags);
```

Since there are still few implementations of these methods in the kernel, the changes are not particularly disruptive. 

Next in the list is support for directly splicing two file descriptors where neither is a pipe. This functionality is not (yet) available to user space via `splice()`, but it is used internally to implement `sendfile()` with the `splice()` mechanism. The direct splicing is implemented using a hidden `pipe_inode_info` structure (i.e. a pipe); it is stored in the new `splice_pipe` field of the task structure, so each process can only have one such connection running at any given time. 

One patch which has not been merged - and will likely wait until 2.6.18 at this point - is [the `tee()` system call](<http://lwn.net/Articles/179434/>): 

```
long tee(int fdin, int fdout, size_t len, unsigned int flags);
```

This call requires that both file descriptors be pipes. It simply connects `fdin` to `fdout`, transferring up to `len` bytes between them. Unlike `splice()`, however, `tee()` does not consume the input, enabling the input data to be read normally later on by the calling process. Jens Axboe provides an example implementation of the user-space `tee` utility, which comes down to a couple of calls: 

```
len = tee(STDIN_FILENO, STDOUT_FILENO, INT_MAX, SPLICE_F_NONBLOCK);
        splice(STDIN_FILENO, out_file, len, 0);
```

The input data will be written both to the standard output and the file represented by `out_file` without ever being copied to or from user space. To be sure of copying the entire input data stream, the application must perform the above calls within a loop, of course; see the full example at the end of [the `tee()` patch](<http://lwn.net/Articles/179434/>). 

This call is quite new, and may well change before it makes it into the mainline. Among other things, it might get a new name, since something as simple as `tee()` may already be in use in a number of applications.
