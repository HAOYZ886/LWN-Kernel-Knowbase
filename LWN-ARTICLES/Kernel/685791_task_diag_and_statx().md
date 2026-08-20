---
title: task_diag and statx()
url: https://lwn.net/Articles/685791/
date: "May 4, 2016"
category: "Filesystems-stat; System calls; proc"
author: "By Jonathan Corbet May 4, 2016"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
May 4, 2016

The interfaces supported by Linux to provide access to information about processes and files have literally been around for decades. One might think that, by this time, they would have reached a state of relative perfection. But things are not so perfect that developers are deterred from working on alternatives; the motivating factor in the two cases studied here is the same: reducing the cost of getting information out of the kernel while increasing the range of information that is available. 

#### task_diag

There is no system call in Linux that provides information about running processes; instead, that information can be found in the `/proc` filesystem. Each process is represented by a directory under `/proc`; that directory contains a directory tree of its own with files providing information on just about every aspect of the process's existence. A quick look at the `/proc` hierarchy for a running `bash` instance reveals 279 files in 40 different directories. Whether one wants to know about control-group membership, resource usage, memory mappings, environment variables, open files, namespaces, out-of-memory-killer policies, or more, there is a file somewhere in that tree with the requisite information. 

There are a lot of advantages to `/proc`, starting with the way it implements the classic Unix "everything is a file" approach. The information is readable as plain text, making it accessible from the command line or shell scripts. To a great extent, the interface is self-documenting, though some parts are more obvious than others. The contents of the `stat` file, for example, require an outside guide to be intelligible. 

There are some downsides to this approach too, though. Accessing a file in `/proc` requires a minimum of three system calls — `open()`, `read()`, and `close()` — and that is after the file has been located in the directory hierarchy. Getting a range of information can require reading several files, with the system-call count multiplied accordingly. Some `/proc` files are expensive to read, and much of the resulting data may not be of interest to the reading process. There has been, to put it charitably, no unifying vision guiding the design of the `/proc` hierarchy, so each file there must be approached as a new parsing problem. It all adds up to a slow and cumbersome interface for applications that need significant amounts of information about multiple processes. 

A possible solution comes in the form of [the `task_diag` patch set](<https://lwn.net/Articles/683371/>) from Andrey Vagin; it adds a binary interface allowing the extraction of lots of process information from the kernel using a single request. The starting point is a file called `/proc/task-diag`, which an interested process must open. That process then uses the netlink protocol to send a message describing the desired information, which can then be read back from the same file. 

The request for information is contained within this structure: 

```
struct task_diag_pid {
    	__u64   show_flags;
    	__u64   dump_strategy;
    	__u32   pid;
        };
```

The `dump_strategy` field tells the kernel which processes are of interest. Its value can be one of `TASK_DIAG_DUMP_ONE` (information about the single process identified by `pid`), `TASK_DIAG_DUMP_THREAD` (get information about all threads of `pid`), `TASK_DIAG_DUMP_CHILDREN` (all children of `pid`), `TASK_DIAG_DUMP_ALL` (all processes in the system) or `TASK_DIAG_DUMP_ALL_THREADS` (all threads in the system). 

The `show_flags` field, instead, describes which information is to be returned for each process. With `TASK_DIAG_SHOW_BASE`, the "base" information will be returned: 

```
struct task_diag_base {
    	__u32   tgid;
    	__u32   pid;
    	__u32   ppid;
    	__u32   tpid;
    	__u32   sid;
    	__u32   pgid;
    	__u8    state;
    	char    comm[TASK_DIAG_COMM_LEN];
        };
```

Other possible flags include `TASK_DIAG_SHOW_CREDS` to get credential information, `TASK_SHOW_VMA` and `TASK_SHOW_VMA_STAT` for information on memory mappings, `TASK_DIAG_SHOW_STAT` for resource-usage statistics, and `TASK_DIAG_SHOW_STATM` for memory-usage statistics. If this interface is merged into the mainline, other options will surely follow. 

The patches have been through a few rounds of review. Presumably something along these lines will eventually be merged, but it is not clear that the level of review required to safely add a new major kernel API has happened. There is also no man page for this feature yet. So it would not be surprising if a few more iterations were required before this one is declared to be ready. 

#### statx()

Information about files in Linux, as with all Unix-like systems, comes via the [`stat()` system call and its variants](<http://man7.org/linux/man-pages/man2/stat.2.html>). Developers have chafed against its limitations for a long time. This system call, being enshrined in the POSIX standard, cannot be extended to return more information. It will likely return information that the calling process doesn't need — a wasted effort that can, for some information and filesystems, be expensive. And so on. For these reasons, an extended `stat()` system call has been a topic of discussion for many years. 

Back in 2010, David Howells [proposed a `xstat()` call](<https://lwn.net/Articles/394298/>) that addressed a number of these problems, but that proposal got bogged down in discussion without being merged. Six years later, David is back with [a new version of this patch](<https://lwn.net/Articles/685519/>). Time will tell if he is more successful this time around. 

The new system call is now called `statx()`; the proposed interface is: 

```
int statx(int dfd, const char *filename, unsigned atflag,
    		  unsigned mask, struct statx *buffer);
```

The file of interest is identified by `filename`; that file is expected to be found in or underneath the directory indicated by the file descriptor passed in `dfd`. If `dfd` is `AT_FDCWD`, the `filename` is interpreted relative to the current working directory. If `filename` is null, information about the file represented by `dfd` is returned instead. 

The `atflag` parameter modifies how the information is collected. If it is `AT_SYMLINK_NOFOLLOW` and `filename` is a symbolic link, information is returned about the link itself. Other `atflag` values include `AT_NO_AUTOMOUNT` to prevent filesystems from being automatically mounted by the request, `AT_FORCE_ATTR_SYNC` to force a network filesystem to update attributes from the server before returning the information, and `AT_NO_ATTR_SYNC` to avoid updating from the server, even at the cost of returning approximate information. That last option can speed things up considerably when querying information about files on remote filesystems. 

The `mask` parameter, instead, specifies which information the caller is looking for. The current patch set has fifteen options, varying from `STATX_MODE` (to get the permission bits) to `STATX_GEN` to get the current inode generation number (on filesystems that have such a concept). That mask appears again in the returned structure to indicate which fields are valid; that structure looks like: 

```
struct statx {
    	__u32	st_mask;	/* What results were written [uncond] */
    	__u32	st_information;	/* Information about the file [uncond] */
    	__u32	st_blksize;	/* Preferred general I/O size [uncond] */
    	__u32	st_nlink;	/* Number of hard links */
    	__u32	st_gen;		/* Inode generation number */
    	__u32	st_uid;		/* User ID of owner */
    	__u32	st_gid;		/* Group ID of owner */
    	__u16	st_mode;	/* File mode */
    	__u16	__spare0[1];
    	__u64	st_ino;		/* Inode number */
    	__u64	st_size;	/* File size */
    	__u64	st_blocks;	/* Number of 512-byte blocks allocated */
    	__u64	st_version;	/* Data version number */
    	__s64	st_atime_s;	/* Last access time */
    	__s64	st_btime_s;	/* File creation time */
    	__s64	st_ctime_s;	/* Last attribute change time */
    	__s64	st_mtime_s;	/* Last data modification time */
    	__s32	st_atime_ns;	/* Last access time (ns part) */
    	__s32	st_btime_ns;	/* File creation time (ns part) */
    	__s32	st_ctime_ns;	/* Last attribute change time (ns part) */
    	__s32	st_mtime_ns;	/* Last data modification time (ns part) */
    	__u32	st_rdev_major;	/* Device ID of special file */
    	__u32	st_rdev_minor;
    	__u32	st_dev_major;	/* ID of device containing file [uncond] */
    	__u32	st_dev_minor;
    	__u64	__spare1[16];	/* Spare space for future expansion */
        };
```

Many of those fields match those found in the classic `struct stat` or are close to them. Times have been split into separate second and nanosecond fields, enabling both high-precision timestamps and year-2038 compliance. The `__spare1` array at the end is meant to allow other types of data to be added in the future. Finally, `st_information` gives general information about the file, including whether it's encrypted, whether it's a kernel-generated file, or whether it's stored on a remote server. 

The only response to this patch set, as of this writing, [came from Jeff Layton](<https://lwn.net/Articles/685807/>), who suggested ""I think we really ought to resist excessive bikeshedding this time around"". If the other developers accept that advice, then it's possible that an enhanced `stat()` interface might just get into the kernel sometime this year. Nobody will be able to complain that this particular change has been rushed.
