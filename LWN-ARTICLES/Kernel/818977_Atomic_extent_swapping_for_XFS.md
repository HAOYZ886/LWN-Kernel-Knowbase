---
title: Atomic extent swapping for XFS
url: https://lwn.net/Articles/818977/
date: "May 1, 2020"
category: "Filesystems-XFS"
author: "By Jonathan Corbet May 1, 2020"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
May 1, 2020

Normally, files exist in a filesystem to keep data contained within them separated; seeing data exchanged directly between files is often a sign of filesystem corruption. There are, however, use cases where it is desirable to be able to perform a controlled swap of data between a pair of files. Darrick Wong has recently posted [a patch set](<https://lwn.net/ml/linux-fsdevel/158812825316.168506.932540609191384366.stgit@magnolia/>) implementing this feature for the XFS filesystem, but also making it available in a general way. 

As it happens, XFS has had a data-swapping capability for some time: the rigorously undocumented `XFS_IOC_SWAPEXT` `ioctl()` command will exchange extents of data in two files. This feature exists for one purpose in particular: defragmentation of filesystems. The [`xfs_fsr` utility](<http://man7.org/linux/man-pages/man8/xfs_fsr.8.html>) does its job by scanning a filesystem for the most highly fragmented files — those that are split up into the largest number of extents. It then creates a new file with a single extent large enough to hold one of the fragmented files and copies the data over. The final step is an `XFS_IOC_SWAPEXT` operation to atomically replace the old file's data blocks with the new, defragmented version. 

It seems, however, that there are other interested users out there. Application developers would like a way to replace some or all of the contents of a file in an atomic and safe way — one which preferably does not leave the file corrupted if the system goes down partway through. Currently such tasks must be handled by creating a temporary file, populating it, and renaming it over the original; this works, but it is a multi-step affair that is hard to get right. 

With a well-implemented "swap range" operation, atomically replacing a portion of a file's contents can be done with a series of operations like: 

  * Create a new file in the same filesystem as the one to be modified, using the `O_TMPFILE` option. 
  * Populate the temporary file with the new data. If only a portion of the file's contents are to be changed, the application can start with an [`FICLONE` `ioctl()` operation](<http://man7.org/linux/man-pages/man2/ioctl_ficlonerange.2.html>) to "copy" the old data in by reference; then only the data to be changed needs to be written. 
  * Perform the swap operation to move the new data into the old file. 
  * Close the temporary file, which will then be deleted by the kernel. 

Other processes working with the file will see either the old or the new data, but they will never see a combination of the two. 

Wong's patch set adds a new `FISWAPRANGE` `ioctl()` command to the virtual filesystem layer, meaning that any filesystem can implement it (though only XFS is implemented so far). An application fills in a structure describing the operation that is to be performed: 

```
struct file_swap_range {
    	__s64		file1_fd;
    	__s64		file1_offset;
    	__s64		file2_offset;
    	__s64		length;	
    	__u64		flags;
    	__s64		file2_ino;
    	__s64		file2_mtime;
    	__s64		file2_ctime;
    	__s32		file2_mtime_nsec;
    	__s32		file2_ctime_nsec;
        };
```

One of the two files is identified by `file1_fd`, which holds a file descriptor; the other file is passed directly as the `ioctl()` call's file-descriptor argument. The normal operation of this call will be to exchange `length` bytes from the first file starting at `file1_offset` with the same number of bytes in the second file starting at `file2_offset`. The range of data to move need not be tied to existing extents in the file, but the offsets must be aligned to the filesystem block size, and in most cases `length` must be a multiple of the block size. Interestingly, the two files can, in fact, be the same file, in which case the two ranges of data (which must not overlap) exchange positions. 

The operation is done in an atomic manner, meaning that the file will not be corrupted in case of a crash. 

The `flags` argument can modify the behavior of this operation in a number of ways. The flags defined in the patch set are: 

  * `FILE_SWAP_RANGE_NONATOMIC`: the application doesn't care about atomic behavior, so the operation can potentially be done in a less expensive way. 
  * `FILE_SWAP_RANGE_FILE2_FRESH`: before performing the swap, the kernel will verify that the second file's inode number matches `file2_ino`, and that its modification times match `file2_mtime`, `file2_mtime_nsec`, `file2_ctime`, and `file_ctime_nsec`. In the absence of an exact match, the call will fail with an `EBUSY` error. This is a way of ensuring that nobody else has modified the file while the exchange was being prepared. 
  * `FILE_SWAP_RANGE_FULL_FILES`: ensure that the exchange operation includes all of the data in both files. In other words, this requires both offsets to be zero and both files to have the same length (which must match the `length` field). 
  * `FILE_SWAP_RANGE_TO_EOF`: data is exchanged from the indicated offsets to the ends of both files. In this case, the `length` field is ignored, and the size of the two ranges to swap may differ. 

The XFS implementation records the operation in the transaction log before actually carrying it out. That provides the necessary robustness in the face of a crash, but it also makes filesystems using this feature incompatible with earlier versions of the XFS code. 

Internally, the patch set adds another member to the ever-growing [`file_operations` structure](<https://elixir.bootlin.com/linux/v5.6.7/source/include/linux/fs.h#L1821>): 

```
int (*swap_file_range)(struct file *file_in, struct file *file_out,
    		           struct file_swap_range *fsr);
```

Any other filesystem can support the `FISWAPRANGE` `ioctl()` by implementing this function. 

This patch set is quite young, to the point that reporting on it is probably violating its privacy rights in some jurisdictions. The review process has not yet begun. It is thus fair to expect that things could change significantly before it finds its way into the mainline, at some point in the not-too-close future. It does appear to be a useful feature, though, so it seems likely to be merged at some point.
