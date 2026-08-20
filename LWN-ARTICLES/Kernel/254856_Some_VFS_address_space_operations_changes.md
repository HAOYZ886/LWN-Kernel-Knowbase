---
title: Some VFS address space operations changes
url: https://lwn.net/Articles/254856/
date: "October 17, 2007"
category: "Filesystems-Virtual filesystem layer"
author: "By Jonathan Corbet October 17, 2007"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 17, 2007

Deeply buried in the 2.6.24 patch stream is a set of significant changes to the VFS layer internal API. The core motivation behind this work is to prevent some deadlock problem which, with the old API, could not be avoided without taking a significant performance hit. Anybody maintaining an out-of-tree filesystem will want to have a look and be prepared to start fixing up their code. 

In the older VFS API, two address space operations are provided by filesystems to support writes to files: 

```
int (*prepare_write)(struct file *file, struct page *page, 
        			 unsigned begin, unsigned end);
        int (*commit_write)(struct file *file, struct page *page, 
        			 unsigned begin, unsigned end);
```

A call to `prepare_write()` notifies the filesystem that the VFS intends to write bytes `begin..end` of `file` into the given `page`. It is then the filesystem's responsibility to make sure that the write will work (allocating blocks if need be) and, if a partial block is to be written, the filesystem should populate `page` with the full block's data. Later on, the call to `commit_write()` tells the filesystem that the data has been copied into `page` and can be committed to disk. 

The problem with this API is that the VFS is expected to pass a locked page into `prepare_write()`. There are a number of scenarios which can lead to attempts to lock that page twice, bringing the system to a halt. To avoid this problem, Nick Piggin has created replacements for `prepare_write()` and `commit_write()`: 

```
int (*write_begin)(struct file *file, struct address_space *mapping,
    		       loff_t pos, unsigned len, unsigned flags,
    		       struct page **pagep, void **fsdata);
        int (*write_end)(struct file *file, struct address_space *mapping,
    		     loff_t pos, unsigned len, unsigned copied,
    		     struct page *page, void *fsdata);
```

There are a number of changes, but the key is the fact that a page is no longer passed into `write_begin()`. Instead, that function should allocate the page itself and return it (locked) to the VFS. The call to `write_end()` indicates that the write is complete; it should unlock the page and update the inode's `i_size` field. 

The new `copied` parameter is also important: it is the number of bytes which were actually copied into the page, which might be smaller than `len` predicted. Some of the possible deadlock scenarios involve the handling of page faults while the destination page is locked; a trivial example is when the data being written to the page is also being read from that page. With the new API, a page fault terminates the copying of the data, allowing the page to be unlocked. The fault can be handled while the destination page is unlocked, avoiding the deadlock problems. 

The possibility of short writes does impose an extra cost on filesystems: any data which may be overwritten must be read in regardless, just in case the write operation ends prematurely. There are times, however, when the VFS knows that writes will go the full length; in particular, writes from buffers which are in kernel space must succeed. When such a write is executed, the VFS will pass the `AOP_FLAG_UNINTERRUPTIBLE` flag to `write_begin()` to let the filesystem know that short writes are not a possibility. 

For now, the `prepare_write()` and `commit_write()` VFS methods are still supported in the kernel. If a filesystem does not provide the newer functions, the older ones will be used. The long-term plan almost certainly involves the removal of those methods, though; they cannot be supported in a way which is simultaneously safe and fast.
