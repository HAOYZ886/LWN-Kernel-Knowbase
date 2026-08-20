---
title: Data temperature in Btrfs
url: https://lwn.net/Articles/398503/
date: "August 3, 2010"
category: "Btrfs; Filesystems-Btrfs"
author: "By Jonathan Corbet August 3, 2010"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
August 3, 2010

Linux, like most other operating systems, has long tried to keep frequently-accessed data in main memory. The cost of fetching a page from disk is high, so every I/O operation which can be eliminated by keeping data in a faster location yields a significant performance improvement. Recently, there has been an increasing level of interest in adding more levels of cache; the result has been patches like [bcache](<http://lwn.net/Articles/394672/>), [Cleancache/Frontswap](<http://lwn.net/Articles/386090/>), [zcache](<http://lwn.net/Articles/397574/>), and more. The latest contribution in this area is a set of patches aimed at enabling multi-level caching within the Btrfs filesystem. 

The [patches](<http://lwn.net/Articles/397643/>), posted by Ben Chociej, are not a complete solution at this time. This code, instead, is meant to add the infrastructure needed to determine which data within a filesystem is "hot"; other work, to be done in the near future, will then be able to make use of this information to determine which data would benefit from being hosted on faster media - on a solid-state storage device, perhaps. The copy-on-write nature of Btrfs, along with its built-in volume management code, should make the implementation of this functionality relatively easy. We should find out in "a few weeks," when the first of these patches is promised; meanwhile, there is some interesting instrumentation work to look at. 

These patches work by hooking into the small number of places in Btrfs where new I/O operations are initiated. Each of these places gets a call to: 

```
void btrfs_update_freqs(struct inode *inode, u64 start, u64 len, 
    			    int create);
```

Where `inode` is the inode for the file being operated on, `start` is the beginning offset (in bytes), `len` is the number of bytes being transferred, and the mildly confusing `create` parameter is nonzero iff the operation is a write. This function maintains two red-black trees; the first, which is filesystem-wide, tracks the "hottest" inodes. For each inode, there is another tree tracking the hottest parts of the file. For each tree, the `btrfs_update_freqs()` call will update the stored parameters with the passed-in values. 

The code tracks six independent parameters: the number of reads, a running average of the time between reads, and the time since the last read - along with the same information for writes. In the end, that information gets passed to a piece of deep magic called `btrfs_get_temp()` which boils those numbers down to a single "temperature" value. Your editor would love to simply provide the formula which is used, but it's not that simple - there's a lot of trickery with magic constants and various provisions against integer overflow problems. For those who would like to figure it out for themselves, here's [the source for `btrfs_get_temp()`](<https://lwn.net/Articles/398511/>). 

There are three new `ioctl()` operations added by the patch set. To get the heat information for a specific file, `BTRFS_IOC_GET_HEAT_INFO` may be used. There are also `BTRFS_IOC_GET_HEAT_OPTS` and `BTRFS_IOC_SET_HEAT_OPTS` for querying and setting the state of heat tracking and (someday) migration of data based on the measured temperature data. A debugfs interface is also provided for those who would like to look at all of the data collected by this instrumentation. 

There has not been a huge response to this patch set so far. The biggest complaint should be somewhat predictable: this capability looks like something which would be useful for many filesystems, so implementing it just for Btrfs looks like working at the wrong level. The virtual filesystem (VFS) layer is well placed to track I/O operations and could manage this kind of data collection. The VFS could also, perhaps, use this data to make better decisions on which pages to keep in the page cache. But, as long as the data is locked up within Btrfs, the VFS layer cannot use it, and it cannot be used to benefit any other filesystems. 

The response to this complaint is that only Btrfs has the multiple device support needed to make use of this data. Dave Chinner [finds that justification unconvincing](<https://lwn.net/Articles/398512/>), saying: 

Why does it even need multiple devices in the filesystem? All the filesystem needs to know is the relative speed of regions of it's block address space and to be provided allocation hints. 

There is often a degree of tension between those who would add features to specific filesystems and those who would rather see that functionality done at the VFS level. As a general rule, widely-useful features benefit from being done in the VFS, where they are more widely used and more closely scrutinized. But, often, an individual filesystem implementation can serve as a useful proof of concept and a place where important lessons are learned. All of which is to say that "hot data tracking" will likely make it into the kernel at some point, but it's not clear whether what is merged will resemble the current patches or not.
