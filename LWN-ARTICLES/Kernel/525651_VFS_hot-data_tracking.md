---
title: "VFS hot-data tracking"
url: https://lwn.net/Articles/525651/
date: "November 20, 2012"
category: "Btrfs; Filesystems-Virtual filesystem layer"
author: "By Jonathan Corbet November 20, 2012"
---

By **Jonathan Corbet**  
November 20, 2012

At any level of the system, from the hardware to high-level applications, performance often depends on keeping frequently-used data in a place where it can be accessed quickly. That is the principle behind hardware caches, virtual memory, and web-browser image caches, for example. The kernel already tries to keep useful filesystem data in the page cache for quick access, but there can also be advantages to keeping track of "hot" data at the filesystem level and treating it specially. In 2010, a [data temperature tracking patch set](<https://lwn.net/Articles/398503/>) for the Btrfs filesystem was posted, but then faded from view. Now the idea has returned as a more general solution. The current form of the patch set, posted by Zhi Yong Wu, is called [hot-data tracking](<https://lwn.net/Articles/525425/>). It works at the virtual filesystem (VFS) level, tracking accesses to data and making the resulting information available to user space via a couple of mechanisms. 

The first step is the instrumentation of the VFS to obtain the needed information. To that end, Zhi Yong's patch set adds hooks to a number of core VFS functions (`__blockdev_direct_IO()`, `readpage()`, `read_pages()`, and `do_writepages()`) to record specific access operations. It is worth noting that hooking at this level means that this subsystem is not tracking data accesses as such; instead, it is tracking operations that cause actual file I/O. The two are not quite the same thing: a frequently-read page that remains in the page cache will generate no I/O; it could look quite cold to the hot-data tracking code. 

The patch set uses these hooks to maintain a surprisingly complicated data structure, involving a couple of red-black trees, that is hooked into a filesystem's `superblock` structure. Zhi Yong used this bit of impressive ASCII art to describe it in the [documentation file](<https://lwn.net/Articles/525652/>) included with the patch set: 

> 

```
>     heat_inode_map           hot_inode_tree
>         |                         |
>         |                         V
>         |           +-------hot_comm_item--------+
>         |           |       frequency data       |
>     +---+           |        list_head           |
>     |               V            ^ |             V
>     | ...<--hot_comm_item-->...  | |  ...<--hot_comm_item-->...
>     |       frequency data       | |        frequency data
>     +-------->list_head----------+ +--------->list_head--->.....
>            hot_range_tree                  hot_range_tree
>                                                  |
>                  heat_range_map                  V
>                        |           +-------hot_comm_item--------+
>                        |           |       frequency data       |
>                    +---+           |        list_head           |
>                    |               V            ^ |             V
>                    | ...<--hot_comm_item-->...  | |  ...<--hot_comm_item-->...
>                    |       frequency data       | |        frequency data
>                    +-------->list_head----------+ +--------->list_head--->.....
>
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

In short, the idea is to track which inodes are seeing the most I/O traffic, along with the hottest data ranges within those inodes. The subsystem can produce a sorted list on demand. Unsurprisingly, this data structure can end up using a lot of memory on a busy system, so Zhi Yong has added a shrinker to clean things up when space gets tight. Specific file information is also dropped after five minutes (by default) with no activity. 

There is a new `ioctl()` command (`FS_IOC_GET_HEAT_INFO`) that can be used to obtain the relevant information for a specific file. The structure it uses shows the information that is available: 

```
struct hot_heat_info {
    	__u64 avg_delta_reads;
    	__u64 avg_delta_writes;
    	__u64 last_read_time;
    	__u64 last_write_time;
    	__u32 num_reads;
    	__u32 num_writes;
    	__u32 temp;
    	__u8 live;
        };
```

The hot-data tracking subsystem monitors the number of read and write operations, when the last operations occurred, and the average period between operations. A [complicated calculation](<https://lwn.net/Articles/525843/>) boils all that information down to a single temperature value, stored in `temp`. The `live` field is an input parameter to the `ioctl()` call: if it is non-zero, the temperature will be recalculated at the time of the call; otherwise a cached, previously-calculated value will be returned. 

The `ioctl()` call does not provide a way to query which parts of the file are the hottest, or to get a list of the hottest files. Instead, the debugfs interface must be used. Once debugfs is mounted, each device or partition with a mounted filesystem will be represented by a directory under `hot_track/` containing two files. The most active files can be found by reading `rt_stats_inode`, while the hottest file ranges can be read from `rt_stats_range`. These are the interfaces that user-space utilities are expected to use to make decisions about, for example, which files (or portions of files) should be stored on a fast, solid-state drive. 

Should a filesystem want to influence how the calculations are done, the patch set provides a structure (called `hot_func_ops`) as a place for filesystem-provided functions to calculate access frequencies, temperatures, and when information should be aged out of the system. In the posted patch set, though, only Btrfs uses the hot-data tracking feature, and it does not override any of those operations, so it is not entirely clear why they exist. The changelog states that support for ext4 and xfs has been implemented; perhaps one of those filesystems needed that capability. 

The patch set has been through several review cycles and a lot of changes have been made in response to comments. The list of things still to be done includes scalability testing, a simpler temperature calculation function, and the ability to save file temperature data across an unmount. If nothing else, some solid performance information will be required before this patch set can be merged into the core VFS code. So hot-data tracking is not 3.8 material, but it may be ready for one of the subsequent development cycles.
