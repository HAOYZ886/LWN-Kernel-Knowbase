---
title: "SEEK_HOLE or FIEMAP?"
url: https://lwn.net/Articles/260795/
date: "December 3, 2007"
category: FIEMAP ioctl; Filesystems
author: "By Jonathan Corbet December 3, 2007"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 3, 2007

Sparse files have an apparent size which is larger than the amount of storage actually allocated to them. The usual way to create such a file is to seek past its end and write some new data; Unix-derived systems will traditionally not allocate disk blocks for the portion of the file past the previous end which was skipped over. The result is a "hole," a piece of the file which logically exists, but which is not represented on disk. A read operation on a hole succeeds, with the returned data being all zeroes. Relatively smart file archival and backup utilities will recognize holes in files; these holes are not stored in the resulting archive and will not be filled if the file is restored from that archive. 

The process of recognizing holes is relatively primitive, though: about the only way to do it in a portable way is to simply look for blocks filled with zeroes. This technique works, but it requires making a pass over the data to obtain information which the lower levels of the system already know. It seems like there should be a better way. 

About two years ago, the Solaris ZFS developers [proposed an extension to `lseek()`](<http://blogs.sun.com/bonwick/entry/seek_hole_and_seek_data>) which would allow an application to find the holes in sparse files more efficiently. This extension works by adding two new "whence" options: 

  * `SEEK_HOLE` positions the file descriptor to the beginning of the first hole which occurs after the given offset. For the purposes of this operation, "hole" is defined as a region of all zeros of any length, but the system is not required to actually detect all holes. So, in practice, small ranges of zeroes will be skipped over, as will, in all likelihood, large (multi-block) ranges which have actually been written to disk. 

  * `SEEK_DATA` moves to the start of next region (after the given offset) which is not a hole. 

This functionality has been part of Solaris for a while; the Solaris developers would like to see it spread elsewhere and become something more than a Solaris-only extension. To that end, Josef Bacik has recently posted [an implementation](<http://lwn.net/Articles/260699/>) of this extension for Linux. Internally, it adds a new member to the `file_operations` structure (`seek_hole_data()`) intended to allow filesystems to efficiently implement the new operations. 

One might argue that anybody who wants to separate holes and data in a file can already do so with the `FIBMAP` `ioctl()` command. While that is true, `FIBMAP` is an inefficient way of getting this sort of information, especially on filesystems which support extents. A `FIBMAP` call returns the mapping information for exactly one block; mapping out a large file may require millions of calls when, once again, the filesystem should already know how to provide that information in a much more straightforward manner. 

Even so, this patch looks relatively unlikely to make it into the mainline. The API is unpopular, being seen as ugly and as a change in the semantics of the `lseek()` call. But, more to the point, it may be interesting to learn much more about the representation of a file than just where the holes are. And, as it turns out, there is already a proposed `ioctl()` command which can provide all of that information. That interface is the [`FIEMAP` `ioctl()`](<https://lwn.net/Articles/260803/>) specified by Andreas Dilger back in October. 

A `FIEMAP` call takes the following structure as an argument: 

```
struct fiemap {
    	__u64	fm_start;	 /* logical starting byte offset (in/out) */
    	__u64	fm_length;	 /* logical length of map (in/out) */
    	__u32	fm_flags;	 /* FIEMAP_FLAG_* flags for request (in/out) */
    	__u32	fm_extent_count; /* number of extents in fm_extents (in/out) */
    	__u64	fm_end_offset;	 /* end of mapping in last ioctl */
    	struct fiemap_extent	fm_extents[0];
        };
```

An application wanting to learn something about how a file is stored will put the starting offset into `fm_start` and the length of the region of interest in `fm_length`. If `fm_flags` contains `FIEMAP_FLAG_NUM_EXTENTS`, the system call will simply set `fm_extent_count` to the number of extents used to store the specified range of bytes and return. In this form, `FIEMAP` can be used to determine how fragmented the file is on disk. 

If the application is looking for more information than that, it will allocate enough space for one or more `fm_extents` structures: 

```
struct fiemap_extent {
        	__u64 fe_offset;/* offset in bytes for the start of the extent */
        	__u64 fe_length;/* length in bytes for the extent */
        	__u32 fe_flags; /* returned FIEMAP_EXTENT_* flags for the extent */
        	__u32 fe_lun;   /* logical device number for extent(starting at 0)*/
        };
```

In this case, `fm_extent_count` should be set to the number of these structures before making the `FIEMAP` call. On return, these structures (as many as is indicated by the returned value of `fm_extent_count`) will be filled in with information on the actual file extents; `fe_offset` says where (on disk) the extent starts, and `fe_length` is the size of the extent. There are quite a few values which can appear in the `fe_flags` field: 

  * `FIEMAP_EXTENT_HOLE` says that there is no data for this range of the file - it's a hole. 

  * `FIEMAP_EXTENT_UNWRITTEN` says that the space has been allocated on disk, but that nothing has been written to that space. Space which has been preallocated with `fallocate()` would be marked this way. 

  * `FIEMAP_EXTENT_UNMAPPED`, instead, marks an extent where some application has written data, but for which no disk blocks have been allocated. 

  * `FIEMAP_EXTENT_DELALLOC` indicates that delayed allocation is being done; this flag implies `FIEMAP_EXTENT_UNMAPPED` as well. 

  * `FIEMAP_EXTENT_SECONDARY` is an indication that the data for this segment is in some sort of secondary storage; one would see this flag on filesystems managed by some sort of hierarchical storage manner. This flag, too, is likely to imply `FIEMAP_EXTENT_UNMAPPED`. 

  * `FIEMAP_EXTENT_NO_DIRECT` says that the data cannot be accessed directly - it requires processing (decompression or decryption, for example) first. 

  * `FIEMAP_EXTENT_LAST` marks the final extent in a file. 

  * `FIEMAP_EXTENT_EOF` indicates that the requested range goes beyond the end of the file. 

  * `FIEMAP_EXTENT_ERROR` marks an extent which has experienced some sort of error; the `fe_offset` field will contain an error number in this case. 

  * `FIEMAP_EXTENT_UNKNOWN` says that the data exists, but its location is unknown. This flag would describe much of your editor's personal file space, though it is unclear how the kernel would know that. 

As can be seen, there is a wealth of information available from this new call, including details on how the file has been split up on disk, allocation strategies, and even the decisions made by a hierarchical storage engine. An implementation exists for the ext4 filesystem. None of this code has been pushed toward the mainline yet, but it would be surprising if that did not happen sometime in the relatively near future. Once that is done, the C library will be able to implement `SEEK_HOLE` and `SEEK_DATA` in user space, should that be desirable.
