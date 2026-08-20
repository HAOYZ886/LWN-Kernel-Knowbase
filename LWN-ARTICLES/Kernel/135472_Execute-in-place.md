---
title: "Execute-in-place"
url: https://lwn.net/Articles/135472/
date: "May 11, 2005"
category: Block layer; Execute in place
author: 
---

Execute-in-place (XIP) support for the Linux kernel has been on the embedded systems wishlist for some time. Such systems usually have the kernel and relevant application images stored in a directly-accessible ROM or flash memory. This memory generally contains a filesystem, and is treated as a disk drive. This mechanism works, but it can be inefficient: running a program from this memory requires that said program first be copied into (usually scarce) RAM. It would be much better if this code could be executed directly out of the flash-based memory. 

Carsten Otte (of IBM) has posted [a set of patches adding XIP support](<https://lwn.net/Articles/135438/>) to the 2.6 kernel. These patches, in addition, enable fast memory-to-memory block I/O for such devices, shorting out the page cache and most of the block layer. As a result, the XIP patches are useful in a number of situations, such as, as Carsten notes, for shared-memory block devices used to communicate between (virtual) systems. 

The first step is to add support at the block driver level. To that end, a new method is added to the `block_device_operations` structure: 

```
int (*direct_access) (struct inode *inode, sector_t sector, 
                              unsigned long *data);
```

This method, if implemented, should come up with a kernel virtual address corresponding to the given `sector` on the block device represented by `inode`. That address, which must remain valid until the device is closed, is returned in `*data`. The return value is zero on success or a negative error code in case of problems. 

The next step is a new method in the `address_space_operations` structure: 

```
struct page *(*get_xip_page)(struct address_space *space, 
                                     sector_t blockno, int create);
```

This method's job is to translate a specific block number within a filesystem to a `page` structure pointing to its directly-mapped data. It is a filesystem-specific function which will translate `blockno` to a sector number on the underlying device, then use that device's `direct_access()` method to get an address. Carsten has posted [an implementation for ext2](<https://lwn.net/Articles/135442/>) which shows how this method can be put together. 

So far, the XIP patches enable fast, memory-to-memory device access, but they do not yet implement true execute-in-place operation. The last step is to replace the usual `nopage()` VMA operation (`filemap_nopage()`) with a new version (`filemap_xip_nopage()`) when the underlying device and filesystem support XIP. The new `nopage()` method will (using `get_xip_page()`) handle page faults by causing a process's page tables to point directly to the on-"disk" pages, rather than reading those pages into RAM. Some other technique will be needed to run the kernel itself in an XIP mode, but anything that is invoked thereafter can be run directly from the memory device. 

Put the above pieces together, and Linux has a complete execute-in-place implementation. Supporting XIP at the block level is not the only way it could be implemented; David Woodhouse [pointed out](<https://lwn.net/Articles/135493/>) that an alternative approach is to use a special-purpose filesystem. Carsten's patches, however, point out a way in which any filesystem could be made to work in an XIP mode.
