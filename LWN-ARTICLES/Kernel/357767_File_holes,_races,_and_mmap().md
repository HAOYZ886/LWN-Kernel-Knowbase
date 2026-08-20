---
title: "File holes, races, and mmap()"
url: https://lwn.net/Articles/357767/
date: "October 21, 2009"
category: Filesystems
author: "October 21, 2009 This article was contributed by Goldwyn Rodrigues"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

October 21, 2009

This article was contributed by Goldwyn Rodrigues

File operations using `truncate()` have always had race conditions. Developers have always been concerned with file writes racing against file size modifications. Various corner cases exist where data could either be lost or ignored when an error occurs or unexpected data may occur where zeros are expected for holes in the file. Jan Kara's [patch](<http://lwn.net/Articles/353240/>) is an attempt to fix such races, and it depends on the [new truncate sequence](<http://lwn.net/Articles/341352/>), which corrects the way the inode size of the file is set. 

####  Holes

A hole in a file is an area represented by zeros. It is created when data is written at an offset beyond the current file size, or the file size is "truncated" to something larger than the current file size. The space between the old file size and the offset (or new file size) is filled by zeros. Most filesystems are smart enough to mark the holes in the inode, and not store them physically on disk (these are also known as sparse files). The filesystem marks blocks in the inode to denote that they are part of a hole. When a user requests data from an offset in a hole, the filesystem creates a page filled with zeroes and passes it to user space. 

The handling of holes becomes a little tricky when the holes are not aligned to the filesystem block boundary. In that case, parts of blocks must be zeroed to represent the holes. For example, a 12k file on a filesystem with 4k block size with a hole at offset 2500 of size 8192, would require the last 1596 (4096-2500) bytes of the first block to be set to zero and the first 2500 bytes of the third block to be set to zeroes. The second block is bypassed in the inode's list of data blocks and does not occupy any space on disk. 

![\[File hole\]](https://static.lwn.net/images/file-hole.png)

#### Mmap

`mmap()` is a system call to map the contents of a file into memory. The call takes the address where the file should be mapped, a file descriptor, the offset within the file to be mapped, and the length of data from the offset to be mapped. Usually, the address passed is NULL, so that the kernel can choose an address and provide it to the process. Mmap can be performed in two ways: 

  * Private mapping - defined by MAP_PRIVATE, this map is private to the process. Any modifications to the data are not reflected to the file. If the process modifies the data, the page is copied and modifications are performed in the new page. This is popularly known as copy-on-write (COW) 
  * Shared mapping - defined by MAP_SHARED, this map can be shared among processes, and can be used as an effective tool for Inter-Process Communication (IPC). Any modification performed in the file are written back to the disk, and is available for other processes to read. However, data writes to disk are not guaranteed to be immediate, and are usually performed when the process calls `msync()` or `munmap()`. 
When a process calls `mmap()`, the kernel sets up Virtual Memory Address (VMA) region to map the pages of the file to disk. It assigns the file's `struct vm_operations` to `vma->vm_ops`. `struct vm_operations` contains pointers to a set of functions which assist in getting the pages to memory on demand. [`vm_operations.fault()`](<http://lwn.net/Articles/242625/>) is called when the user access a virtual memory area not present in main memory. It is responsible for fetching the page from disk and putting it into memory. If the vma is shared, `vm_operations.page_mkwrite()` makes the page writable, otherwise the page is duplicated using COW. `page_mkwrite()` is responsible for keeping track of all information required by the filesystem, such as buffer_heads, to put the data back on disk. Typically, this means preparing the block for write, checking that there is enough disk space (returning `ENOSPC` if not), and committing the write. 

The current sequence in `page_mkwrite()` can race with file size changes performed by `truncate()`. File truncates happening while the data is written back from a shared `mmap()` could lead to unexpected results, such as loss of data or data in places where zeros are expected. 

#### Data loss

Data loss in a program can occur in a specific case where a program maps a file into memory bigger than the current file size. To explain how data loss can occur, consider the following code snippet for writing a file, on a system with a block size of 1024 bytes and a page size of 4096 bytes: 

```
ftruncate(fd, 0);
        pwrite(fd, buf, 1024, 0);
    
        map = mmap(NULL, 4096, PROT_WRITE, MAP_SHARED, fd, 0);
        map[0] = 'a';  /* page_mkwrite() for index 0 is called */
```

Note that even though the file size is set to 1024 bytes, the map is mapped to 4096, which is beyond the current file size. This is feasible because pages from a file are mapped in page size chunks. Since there is a change to the shared memory, this causes the entry in the page table to become writable. 

```
pwrite(fd, buf, 1, 10000);
        map[3000] = 'b';
        fsync(fd); /* writepage() for index 0 is called */
```

When the first `page_mkwrite()` is called, only block 0 is allocated because the file size can fit in 1024 bytes. However, when the program later increases the file size and calls `fsync()`, the `writepage()` needs to allocate 3 more blocks to complete the write caused by changing `map[3000]`. In that situation, if the user's quota exhausts or the filesystem has no more space, the data modified by `map[3000]` is silently ignored. 

#### Unexpected non-zeroes in a hole

A non-zero character can end up in a hole if the process dies after extending the file, but before zeroing the page and writing it. To understand the problem, consider the following code snippet: 

```
ftruncate(fd, 1000);
        map = mmap(NULL, 4096, PROT_WRITE, MAP_SHARED, fd, 0);
        while (1)
            map[1020] = 'a';
```

The program continuously writes at offset 1020. The kernel zeroes the page from offset 1000 to 4096 before writing the page to disk. However, `map[1020]` can be set after the kernel has zeroed the page. The page is unlocked and set for write-back. In this case, a non-zero character will be written to the disk. This is not a problem because it is out of the range of the file size. However, if another process increases the file size (and thus the size of the hole), and is killed before re-zeroing and writing the page, the "dirty character" will be included in the file the next time the file is read. This problem exists regardless of the block size of the filesystem. The complete program to demonstrate this problem is posted [here](<http://news.gmane.org/find-root.php?message_id=<20090616144217.GA18063%40duck.suse.cz>>). 

#### Solution

Jan's patch introduces helper functions which facilitate the creation of holes: `block_prepare_hole()` and `block_finish_hole()`. These functions are respectively called in `write_begin()` and `write_end()` sequence of address space operations if the current file position is detected to be beyond the current file size, that is, for creation of a hole. `write_begin()` and `write_end()` are usually called in `page_mkwrite()`. The part of the page in the hole is zeroed in `block_prepare_hole()` instead of `block_write_full_page()`. The page remains locked during the entire `page_mkwrite()` sequence, so it is protected against writes from other processes. The truncate operation can only occur once the page lock is released, serializing the sequence. This resolves the problem of the stray data that can land in the hole. 

On the other hand, `block_finish_hole()` is responsible for marking the part of the page in the hole as read-only. If the process attempts to write anything in the part of the hole belonging to the page, `page_mkwrite()` will be called. The kernel gets an opportunity to allocate `buffer_heads`, if required, for the additional write, or return an error in the case of `ENOSPC` or `EDQUOT`. If there is an error, `write_begin()` will return it, thus, modifying the mapped memory area, will return an error (`SIGSEGV`). The function to write data back to disk, `block_write_full_page()`, checks for all pages' buffers in the page instead of just those within the file size, which are delayed or mapped. The new truncate sequence guarantees that the file is not truncated while this is performed. This resolves the problem of data loss. 

The patch introduces a new field `new_writepage` in `struct address_space_operations`, to store the new method used to perform the `writepage()`. Like the new truncate sequence, this field is a temporary hack and will go away once all filesystems adhere to the new standards of writing the pages to disk. Filesystems implementing the new method of `writepage` must set the `new_writepage` and handle blocks with holes, by preparing the creation of holes in `write_begin()`, and to terminate it in `write_end()`. The old behavior of handling `page_mkwrite()` is restored in `noalloc_page_mkwrite()`. It does not allocate any blocks on page fault and marks all the unmapped buffers in the page as delayed so that `block_write_full_page()` writes them. 

`simple_create_hole()` is a new function analogous to the rest of the `simple_*` functions; it is a simple way of creating hole in a file. The function zeros out the part of the pages which are a part of the hole. This function is called whenever file size is truncated beyond the current file size. 

This posting is the third revision of the patch, and most of the objections have been ironed out in the earlier two passes. Since this patch deals with closing a race condition, it is probable that it will be included eventually. However, this series depends on the new truncate series, so it must wait for those patches to be incorporated in the mainline kernel. Moreover, the hackish method of distinguishing the new writepage must be removed. This requires all filesystems transition to using the new writepage sequence. 

[ Thanks to Jan Kara for reviewing the article. ]
