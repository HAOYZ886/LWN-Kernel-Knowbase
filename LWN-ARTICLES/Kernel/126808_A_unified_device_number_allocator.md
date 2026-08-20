---
title: A unified device number allocator
url: https://lwn.net/Articles/126808/
date: "March 8, 2005"
category: Device numbers
author: 
---

Traditionally, device drivers have added their devices to the system with calls to `register_chrdev()` or `register_blkdev()`. These functions served two functions: allocating a portion of the device number space, and making specific devices available to user space. In 2.6, things changed a bit. For character devices, `register_chrdev()` was replaced by the combination of `alloc_chrdev_region()`, which allocates device numbers, and `cdev_add()`, which attaches a device to a specific number. On the block side, `register_blkdev()` has become optional, but it can still be used to allocate a block major number. The association of block devices with numbers is done with add_disk(). 

In other words, the allocation of device number space and the association of specific numbers with devices have been split in the 2.6 kernel. Matt Mackall was looking at the allocation side recently, where he noticed a fair amount of duplicated code between the char and block implementations. The current code is also unable to perform dynamic allocation of major numbers outside of the traditional 0..255 range. So Matt put together [a patch](<https://lwn.net/Articles/126743/>) which cleans things up a bit. 

The new allocation scheme relies on simple linked lists. When a new device number request comes in, the code searches the (sorted) list to see if the request can be satisfied. If so, a new entry is added to the list, and the starting device number is returned. This work is done by the new function `register_dev()`: 

```
int register_dev(dev_t base, dev_t top, int size, const char *name,
                         struct list_head *list, dev_t *ret);
```

This function requests that a range of `size` numbers be allocated from the given `list`. The first number should fall between `base` and `top`; if a suitable range is found, that first number will be returned in `ret`. The `list` is a simple, `list_head` structure which is initially empty; the caller must provide locking to prevent concurrent calls to `register_dev()` using the same `list`. 

The new interface works; it also replaces a fair amount of common code in the char and block code. Other than some quibbles about potential performance problems resulting from the linear list search algorithm (which should not really matter, since device number allocation is a rare operation), there seem to be no real objections to the new scheme. So it may find its way into a -mm kernel before too long. 

A future change would allow the dynamic allocation of device numbers in the expanded range; for now, dynamic major numbers are allocated from 254 in descending order, as has been done for many years. The patch also retains the `register_chrdev()` and `register_blkdev()` interfaces in a compatibility mode - even though both were essentially obsolete even before the change. At some point in the future, there may be an attempt to deprecate those interfaces; that move would force changes in a great many drivers.
