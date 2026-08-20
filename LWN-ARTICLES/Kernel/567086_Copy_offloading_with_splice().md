---
title: Copy offloading with splice()
url: https://lwn.net/Articles/567086/
date: "September 18, 2013"
category: splice
author: "By Jonathan Corbet September 18, 2013"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
September 18, 2013

One of the most common things to do on a computer is to copy a file, but operating systems have traditionally offered little in the way of mechanisms to accelerate that task. The `cp` program can replicate a filesystem hierarchy using links — most useful for somebody wanting to work with multiple kernel trees — but that trick speeds things up by not actually making copies of the data; the linked files cannot be modified independently of each other. When it is necessary to make an independent copy of a file, there is little alternative to reading the whole thing through the page cache and writing it back out. It often seems like there should be a better way, and indeed, there might just be. 

Contemporary systems often have storage mechanisms that could speed copy operations. Consider a filesystem mounted over the network using a protocol like NFS, for example; if a file is to be copied to another location on the same server, doing the copy on the server would avoid a lot of work on the client and a fair amount of network traffic as well. Storage arrays often operate at the file level and can offload copy operations in a similar way. Filesystems like Btrfs can "copy" a file by sharing a single copy of the data between the original and the copy; since that sharing is done in a copy-on-write mode, there is no way for user space to know that the two files are not completely independent. In each of these cases, all that is needed is a way for the kernel to support this kind of accelerated copy operation. 

Zach Brown has recently posted [a patch](<https://lwn.net/Articles/566263/>) showing how such a mechanism could be added to the `splice()` system call. This system call looks like: 

```
ssize_t splice(int fd_in, loff_t *off_in, int fd_out, loff_t *off_out,
        		   size_t len, unsigned int flags);
```

Its job is to copy `len` bytes from the open file represented by `fd_in` to `fd_out`, starting at the given offsets for each. One of the key restrictions, though, is that one of the two file descriptors must be a pipe. Thus, `splice()` works for feeding data into a pipe or for capturing piped data to a file, but it does not perform the simple task of copying one file to another. 

As it happens, the machinery that implements `splice()` does not force that limitation; instead, the "one side must be a pipe" rule comes from the history of how the `splice()` system call came about. Indeed, it already does file-to-file copies when it is invoked behind the scenes from the `sendfile()` system call. So there should be no real reason why `splice()` would be unable to do accelerated file-to-file copies. And that is exactly what Zach's patch causes it to do. 

That patch set comes in three parts. The first of those adds a new flag (`SPLICE_F_DIRECT`) allowing users to request a direct file-to-file copy. When this flag is present, it is legal to provide values for both `off_in` and `off_out` (normally, the offset corresponding to a pipe must be NULL); when an offset is provided, the file will be positioned to that offset before the copying begins. After this patch, the file copy will happen without the need to copy any data in memory and without filling up the page cache, but it will not be optimized in any other way. 

The second patch adds a new entry to the ever-expanding `file_operations` structure: 

```
ssize_t (*splice_direct)(struct file *in, loff_t off_in, struct file *out, 
    			     loff_t off_out, size_t len, unsigned int flags);
```

This optional method can be implemented by filesystems to provide an optimized implementation of `SPLICE_F_DIRECT`. It is allowed to fail, in which case the `splice()` code will fall back to copying within the kernel in the usual manner. 

Here, Zach worries a bit in the comments about how the `SPLICE_F_DIRECT` flag works: it is used to request both direct file-to-file copying and filesystem-level optimization. He suggests that the two requests should be separated, though it is hard to imagine a situation where a developer who went to the effort to use `splice()` for a file-copy operation would _not_ want it to be optimized. A better question, perhaps, is why `SPLICE_F_DIRECT` is required at all; a call to `splice()` with two regular files as arguments would already appear to be an unambiguous request for a file-to-file copy. 

The last patch in the series adds support for optimized copying to the Btrfs filesystem. In truth, that support already exists in the form of the `BTRFS_IOC_CLONE` `ioctl()` command; Zach's patch simply extends that support to `splice()`, allowing it to be used in a filesystem-independent manner. No other filesystems are supported at this point; that work can be done once the interfaces have been nailed down and the core work accepted as the right way forward. 

Relatively few comments on this work have been posted as of this writing; whether that means that nobody objects or nobody cares about this functionality is not entirely clear. But there is an ongoing level of interest in the idea of optimized copy operations in general; see [the lengthy discussion of the proposed `reflink()` system call](<https://lwn.net/Articles/333783/>) for an example from past years. So, sooner or later, one of these mechanisms needs to make it into the mainline. `splice()` seems like it could be a natural home for this type of functionality.
