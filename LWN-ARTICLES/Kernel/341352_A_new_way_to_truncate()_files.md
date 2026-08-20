---
title: A new way to truncate() files
url: https://lwn.net/Articles/341352/
date: "July 15, 2009"
category: "Filesystems-Virtual filesystem layer"
author: "July 15, 2009 This article was contributed by Goldwyn Rodrigues"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

July 15, 2009

This article was contributed by Goldwyn Rodrigues

Changes are happening the way the virtual filesystem and virtual memory subsystems interact. One of the goals of this work is to close a race existing in `page_mkwrite()` (called when a previously read-only page table entry (PTE) is about to become writable), namely to make sure that file blocks are properly zero-filled if a truncate operation increases the file size. As a part of improving `page_mkwrite()`, Jan Kara posted a set of [patches](<http://lwn.net/Articles/337548/>). However, these patches introduced a new lock to resolve the problem. Nick Piggin thinks this can be done by using the page lock instead of a new lock. As a first step toward a resolution of the problem, he posted a set of [patches](<http://thread.gmane.org/gmane.linux.file-systems/33900>) to improve the truncate sequence. The new truncate sequence is simpler to understand, flexible in usage, and most important of all, handles errors gracefully. 

The `truncate()` and `ftruncate()` system calls are used to set a file to the specified size. If the file is larger than the argument passed, the file size is truncated and the part of the file greater than the passed file size is lost. If the current file size is smaller than the passed file size argument, the file size is increased and the area greater than the previous file size is filled with zeros. The file size argument passed cannot be greater than the maximum possible file size on the filesystem. 

A user space `truncate()` call is handled, inside the kernel, by `do_sys_truncate()`, which is responsible for weeding out all error cases (such as "the inode is a directory" or permission errors). It breaks leases of files locked with `flock()` and calls `do_truncate()`. `do_truncate()`, in turn, creates a new attribute structure with the new length of the file and calls `notify_change()` with the `dentry` and the new attributes under `inode->i_mutex`. `notify_change()` calls the generic `inode_setattr()`, either explicitly, or through the filesystem implementation of `setattr()`. Then, `inode_setattr()` calls `vmtruncate()` to set the inode size and unmap the pages mapped beyond the new file size. After unmapping the pages, the associated filesystem's `truncate()` operation is called to free the disk blocks associated with the file. 

According to Nick, this approach has problems: 

Big problem with the previous calling sequence: the filesystem is not called until i_size has already changed. This means it is not allowed to fail the call, and also it does not know what the previous i_size was. Also, generic code calling vmtruncate to truncate allocated blocks in case of error had no good way to return a meaningful error (or, for example, atomically handle block deallocation). 

Nick's new truncate sequence introduces a way to better communicate error conditions and consolidates the checks which most filesystems currently perform individually. The [original intention](<http://lwn.net/Articles/340109/>) was to add a new `truncate()` operation in `struct inode_operations` which would be called directly for a truncate operation in `inode_setattr()`. Christoph Hellwig [ disagreed](<http://news.gmane.org/find-root.php?message_id=%3c20090706172241.GA26042%40infradead.org%3e>) with the call sequence, stating that the new truncate function should be called from notify_sequence, and not from inode_setattr which is the default implementation for `inode_operations.setattr`. Nick felt that clearing `ATTR_SIZE` before calling generic `setattr` is not unusual (discussed later), so he decided to introduce his changes with a flag called `new_truncate` in `struct inode_operations`, and not using a new truncate function altogether. The `new_truncate` flag indicates that the `truncate()` function in the inode operations handles the new format. Nick admits that this is a nasty hack when he introduces the variable in `inode_operations`. However, it will be required until all filesystems transition to the new truncate sequence. Filesystem code which does not implement the new convention will automatically initialize `new_truncate` to zero, indicating that it has not transitioned yet. 

The [first patch](<http://news.gmane.org/find-root.php?message_id=%3c20090710073230.599337765%40suse.de%3e>) in the patch series introduces new functions to facilitate the change. `inode_newsize_ok()` performs simple checks to check if the intended new file size is within limits defined by the filesystem or is not a swap file: 

```
int inode_newsize_ok(struct inode *inode, loff_t offset)
```

These checks are currently done by individual filesystems. Using this function results in cleanups in individual filesystem code. 

The `truncate_pagecache()` function truncates the inode pages and unmaps the pages in the range beyond the new filesystem size: 

```
void truncate_pagecache(struct inode *inode, loff_t old, loff_t new);
```

`truncate_pagecache()` should ideally be called before the filesystem releases the data blocks associated with the inode. This way the page cache will always be in sync with the on-disk format and the filesystem will not have to deal with situations such as `writepage()` being called for a page that has had its underlying blocks deallocated. 

The `vmtruncate()` function is consolidated for NUMA and non-NUMA architectures in `mm/truncate.c`. However, `vmtruncate()` is deprecated. Instead, `truncate_pagecache()` and `inode_newsize_ok()` introduced in the first patch should be used. 

The third patch is the main patch of the series which uses the new truncate operation. It introduces `simple_setsize()`, which performs equivalent of `vmtruncate()`. `simple_setsize()` is called by `inode_setattr()` when `ATTR_SIZE` is passed. So filesystems implementing their own truncate code in setattr must clear `ATTR_SIZE` before calling the generic `inode_setattr()`. 

To follow the new standards of the truncate operation, individual filesystems must implement their own `setsize()` function, which performs the file size validation checks, truncates the page cache, and truncates the data blocks associated with the inode. Filesystems must not trim off blocks past `i_size` using `vmtruncate()`. Instead, they must handle the truncate in the filesystem code using `truncate_pages()`. This creates a better opportunity to catch errors. The `inode_operations.new_truncate` and `inode_operations.truncate` fields will go away once all filesystems are converted. 

To demonstrate the change, the final patch in the series modifies the ext2 filesystem to use the new truncate interface. The patch introduces `ext_setsize()` to set the inode size of the file, truncate the pagecache, and, finally, trim the data blocks on the filesystem. If `ATTR_SIZE` is set, `ext2_setattr()` calls `ext2_setsize()` to perform the truncate and the `ATTR_SIZE` is unset so that inode_setattr() does not perform the operations again. 

The new truncate patchset has gone through a fair share of review and is pretty likely to get merged. However, it would require the "nasty hack" until all filesystems have transitioned to the new way of truncating files, after which the hack will be removed. The patches are part of the improvements Nick wants to see in the VM layer. Based on the new truncate patches, Nick posted an [RFC](<http://thread.gmane.org/gmane.linux.file-systems/33910>) on how he would close a race condition in `page_mkwrite` when a file is truncated beyond the current file size. Closing races is a good thing, so expect this work to proceed apace.
