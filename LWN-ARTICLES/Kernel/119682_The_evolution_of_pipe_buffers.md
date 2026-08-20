---
title: The evolution of pipe buffers
url: https://lwn.net/Articles/119682/
date: "January 18, 2005"
category: Circular buffers; Pipes; splice
author: 
---

[Last week](<http://lwn.net/Articles/118750/>), this page looked at the new circular buffer structure used to implement Unix pipes in 2.6.11-rc1, and noted that the plan was to evolve that structure into something more general. Since then, Linus has taken a couple more steps; it must be time to catch up. 

One change which has already been merged is the addition of a set of operations for pipe buffers: 

```
struct pipe_buf_operations {
    	int can_merge;
    	void *(*map)(struct file *, struct pipe_inode_info *, 
                         struct pipe_buffer *);
    	void (*unmap)(struct pipe_inode_info *, struct pipe_buffer *);
    	void (*release)(struct pipe_inode_info *, struct pipe_buffer *);
        };
```

The `can_merge` flag addresses one of the issues raised last week: coalescing of writes into existing pages in the buffer. If `can_merge` is non-zero, coalescing will be performed. Otherwise, each write to a pipe buffer will result in the creation of a new circular buffer entry, and, by default, the allocation of a new page. 

The `map()` and `unmap()` methods are charged with controlling the visibility of pipe buffer pages in the kernel's virtual address space. The default `map()` operations for buffers implementing Unix pipes is quite simple: 

```
static void *anon_pipe_buf_map(struct file *file, 
                                       struct pipe_inode_info *info, 
                                       struct pipe_buffer *buf)
        {
                return kmap(buf->page);
        }
```

Since the mapping operation has been abstracted out, there are now fewer assumptions regarding how data is really stored within a pipe buffer. This opens the door to different pipe implementations, such as pipes which implement a direct window into device memory. 

The `release()` method should clean things up when the pipe buffer is no longer needed. 

Linus has also created [an initial implementation of a `splice()` system call](<https://lwn.net/Articles/119680/>), though this work is clearly not ready for merging at this point. This system call looks like: 

```
long sys_splice(int fdin, int fdout, size_t len, unsigned long flags);
```

`fdin` and `fdout` are two file descriptors; a call to `sys_splice()` will result in `len` bytes being copied from `fdin` to `fdout`, one of which is expected to be a pipe. The `flags` argument is not currently used by the sample implementation. 

To make `sys_splice()` work, Linus added two new methods to the ever-expanding `file_operations` structure: 

```
ssize_t (*splice_write)(struct inode *in_pipe, struct file *out, 
                                size_t len, unsigned long flags);
        ssize_t (*splice_read)(struct file *in, struct inode *out_pipe, 
                               size_t len, unsigned long flags);
```

The patch includes a generic `splice_read()` implementation suitable for filesystem-backed file descriptors. It simply populates the page cache with some pages from the file, then loads those pages into the pipe buffer represented by `out_pipe`. Like ordinary `read()` and `write()` methods, the splice variants can transfer fewer bytes than requested. Linus's version will stop at the maximum capacity of a pipe buffer - 16 pages, currently. 

As Linus acknowledges, there are a number of shortcomings to the current implementation - it is incomplete, the interfaces are ugly, and it will oops the system if anything goes wrong. It is, however, an indication of where he expects this work will lead. Stay tuned.
