---
title: "Page-based direct I/O"
url: https://lwn.net/Articles/348719/
date: "August 25, 2009"
category: "Block layer-Direct IO"
author: "By Jonathan Corbet August 25, 2009"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
August 25, 2009

An "address space" in kernel jargon is a mapping between a range of addresses and their representation in an underlying filesystem or device. There is an address space associated with every open file; any given address space may or may not be tied to a virtual memory area in a process's virtual (memory) address space. In a typical process, a number of address spaces will exist for mappings of the executable being run, files the process has open, and ranges of anonymous user memory (which use swap as their backing store). There are a number of ways for processes to operate on their address spaces, one of the stranger of which being direct I/O. A new patch series from Jens Axboe looks to rationalize the direct I/O path a bit, making it more flexible in the process. 

The idea behind direct I/O is that data blocks move directly between the storage device and user-space memory without going through the page cache. Developers use direct memory for either (or both) of two reasons: (1) they believe they can manage caching of file contents better than the kernel can, or (2) they want to avoid overflowing the page cache with data which is unlikely to be of use in the near future. It is a relatively little-used feature which is often combined with another obscure kernel capability: asynchronous I/O. The biggest consumers, by far, of this functionality are large relational database systems, so it is not entirely surprising that a developer currently employed by Oracle is working in this area. 

When the kernel needs to do something with an address space, it usually looks into the associated `address_space_operations` structure for an appropriate function. So, for example, normal file I/O are handled with: 

```
int (*writepage)(struct page *page, struct writeback_control *wbc);
        int (*readpage)(struct file *filp, struct page *page);
```

As with the bulk of low-level, memory-oriented kernel operations, these functions operate on `page` structures. When memory is managed at this level, there is little need to worry about whether it is user-space or kernel memory, or whether it is in the high-memory zone. It's all just memory. The function which handles direct I/O looks a little different, though: 

```
ssize_t (*direct_IO)(int rw, struct kiocb *iocb, const struct iovec *iov,
    			 loff_t offset, unsigned long nr_segs);
```

The use of the `kiocb` structure shows the assumption that direct I/O will be submitted through the asynchronous I/O path. Beyond that, though, the `iovec` structure pointing to the buffers to be transferred comes directly from user space, and it contains user-space addresses. That, in turn implies that the `direct_IO()` function must itself deal with the process of getting access to the user-space buffers. That task is generally handled in VFS-layer generic code, but there's another problem: the `direct_IO()` function cannot be called on kernel memory. 

The kernel does not normally need to use the direct I/O paths itself, but there is one exception: the loopback driver. This driver allows an ordinary file to be mounted as if it were a block device; it can be most useful for accessing filesystem images stored within disk files. But files accessed via a loopback mount may well be represented in the page cache twice: once on each side of the loopback mount. The result is a waste of memory which could probably be put to better uses. 

It would, in summary, be nice to change the `direct_IO()` interface to avoid this memory waste, and to make it a little bit more consistent with the other address space operations. That is what [Jens's patch](<http://lwn.net/Articles/347371/>) does. With that patch, the interface becomes: 

```
struct dio_args {
    	int rw;
    	struct page **pages;
    	unsigned int first_page_off;
    	unsigned long nr_segs;
    	unsigned long length;
    	loff_t offset;
    
    	/*
    	 * Original user pointer, we'll get rid of this
    	 */
    	unsigned long user_addr;
        };
    
        ssize_t (*direct_IO)(struct kiocb *iocb, struct dio_args *args);
```

In the new API, many of the relevant parameters have been grouped into the `dio_args` structure. The memory to be transferred can be found by way of the `pages_array`. The higher-level VFS direct I/O code now handles the task of mapping user-space buffers and creating the `pages` array. 

The impact on the code is, for the most part, small; it's mostly a matter of moving the location where the translation from user-space address to `page` structures is done. The current code does have a potential problem in that it only processes one I/O segment at a time, possibly creating performance problems for some kinds of applications. That mode of operation is not really wired into the system, though, and can presumably be fixed at some point. 

The only other [objection](<https://lwn.net/Articles/348733/>) came from Andrew Morton, who does not like the way Jens implemented the process of working through the array of `page` structures. The index into this array (called `head_page`) is built into `struct dio` and hidden from the code which is actually working through the pages; that leads to potential confusion, especially if the operation aborts partway through. Andrew called it ""a disaster waiting to happen"" and recommended that indexing be made explicit where the `pages` array is processed. 

That is a detail, though - albeit a potentially important one. The core goals and implementation appear to have been received fairly well. It seems highly unlikely that this code could be ready for the 2.6.32 merge window, but we might see it aiming for the mainline in a subsequent development cycle.
