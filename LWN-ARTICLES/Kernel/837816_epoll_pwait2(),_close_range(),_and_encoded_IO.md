---
title: "epoll_pwait2(), close_range(), and encoded I/O"
url: https://lwn.net/Articles/837816/
date: "November 20, 2020"
category: "Btrfs; Epoll; Filesystems-Btrfs; System calls-close range"
author: "By Jonathan Corbet November 20, 2020"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 20, 2020

The various system calls and other APIs that the kernel provides for access to files and filesystems has grown increasingly comprehensive over the years. That does not mean, though, that there is no need or room for improvement. Several relatively small additions to the kernel's filesystem-related API are under consideration in the development community; read on for a survey of some of this work. 

#### Higher-resolution `epoll_wait()` timeouts

The kernel's "epoll" subsystem provides a high-performance mechanism for a process to wait on events from a large number of open file descriptors. Using it involves creating an epoll file descriptor with [`epoll_create()`](<https://man7.org/linux/man-pages/man2/epoll_create.2.html>), adding file descriptors of interest with [`epoll_ctl()`](<https://man7.org/linux/man-pages/man2/epoll_ctl.2.html>), then finally waiting on events with [`epoll_wait()` or `epoll_pwait()`](<https://man7.org/linux/man-pages/man2/epoll_wait.2.html>). When waiting, the caller can specify a timeout as an integer number of milliseconds. 

The epoll mechanism was added during the 2.5 development series, and became available in the 2.6 release at the end of 2003. Nearly 20 years ago, when this work was being done, a millisecond timeout seemed like enough resolution; the kernel couldn't reliably do shorter timeouts in any case. In 2020, though, one millisecond can be an eternity; there are users who would benefit from much shorter timeouts than that. Thus, it seems it is time for another update to the epoll API. 

Willem de Bruijn duly showed up with [a patch set](<https://lwn.net/ml/linux-kernel/20201116161001.1606608-1-willemdebruijn.kernel@gmail.com/>) adding nanosecond timeout support to `epoll_wait()`, but it took a bit of a roundabout path. Since there is no "flags" argument to `epoll_wait()`, there is no way to ask for high-resolution timeouts directly. So the patch set instead added a new flag (`EPOLL_NSTIMEO`) to `epoll_create()` (actually, to `epoll_create1()`, which was added in 2.6.27 since `epoll_create()` also lacks a "flags" argument). If an epoll file descriptor was created with that flag set, then the timeout value for `epoll_wait()` would be interpreted as being in nanoseconds rather than milliseconds. 

Andrew Morton, however, [complained](<https://lwn.net/ml/linux-kernel/20201116120445.7359b0053778c1a4492d1057@linux-foundation.org/>) about this API. Having one system call set a flag to change how arguments to a different system call would be interpreted was ""not very nice"" in his view; he suggested adding a new system call instead. After a bit of back and forth, that is what happened; the [current version of the patch set](<https://lwn.net/ml/linux-kernel/20201118144617.986860-1-willemdebruijn.kernel@gmail.com/>) adds `epoll_pwait2()`: 

```
int epoll_pwait2(int fd, struct epoll_event *events, int maxevents,
                         const struct timespec *timeout, const sigset_t *sigset);
```

In this version, the `timeout` is passed as a [`timespec` structure](<https://elixir.bootlin.com/linux/v5.9.9/source/include/uapi/linux/time.h#L11>), which includes a field for nanoseconds. 

There has been some discussion of the implementation of this system call, but not a lot of comments on the API, so perhaps this work will go forward in this form. Your editor cannot help but note, however, that this system call, too, lacks a "flags" argument, so the eventual need for an `epoll_pwait3()` can be readily foreseen. 

#### `close_range()` — eventually

The [`close_range()`](<https://lwn.net/Articles/789023/>) system call was added in the 5.9 release as a way to efficiently close a whole list of file descriptors: 

```
int close_range(int first, int last, unsigned int flags);
```

This call will close all file descriptors between `first` and `last`, inclusive. There is currently one `flags` value defined: `CLOSE_RANGE_UNSHARE`, which causes the indicated range of file descriptors to be unshared from any other processes (and does _not_ close them). 

In [this patch set](<https://lwn.net/ml/linux-kernel/20201118104746.873084-1-gscrivan@redhat.com/>), Giuseppe Scrivano adds another flag, `CLOSE_RANGE_CLOEXEC`. This flag will set the "close on `exec()`" flag on each of the indicated file descriptors. Once again, `close_range()` does not actually close the files in this case; it simply marks them to be closed if and when the calling process does an `exec()` in the future. This is, presumably, faster than executing a loop and setting the flag with [`fcntl()`](<https://man7.org/linux/man-pages/man2/fcntl.2.html>) on each file descriptor individually. 

The functionality seems useful, and there have not really been any complaints about the API (there were some issues with the implementation in previous versions of the patch set). Given that `close_range()` is taking on more functionality that does not involve actually closing files, though, it seems increasingly clear that this system call is misnamed. It has only been available since the 5.9 release on October 11, so there are not yet C-library wrappers for it in circulation. So there _is_ time to come up with a better name for this system call, should the desire to do so arise. 

#### Encoded I/O

Some filesystems have the ability to compress and/or encrypt data written to files. Normally, this data will be restored to its original form when read from those files, so users may be entirely unaware that this transformation is taking place at all. What if, however, somebody wanted the ability to work with this "encoded" data directly, bypassing the processing steps within the filesystem code? Omar Sandoval has [a patch set](<https://lwn.net/ml/linux-fsdevel/cover.1605723568.git.osandov@fb.com/>) making that possible. 

The main motivation for this work appears to be backups and, in particular, the transmission of partial or full filesystem images with [the Btrfs `send` and `receive` operations](<https://lwn.net/Articles/581558/>). The whole point of using this mechanism is to create an identical copy of a Btrfs subvolume on another device. If the subvolume is using compression, a `send` will currently decompress the data, which must then be recompressed on the `receive` side, ending up in its original form. If there is a lot of data involved, this is a somewhat wasteful operation; it would be more efficient to just transmit the compressed data. 

With this patch set applied, it becomes possible to read the compressed and/or encrypted data directly and write it directly, with no intervening processing. The first step is to open the subvolume with the new `O_ALLOW_ENCODED` flag. The `CAP_SYS_ADMIN` capability is needed to open a subvolume in this mode; imagine what could happen if an attacker were to write corrupt compressed data to a file, for example. Dave Chinner [argued](<https://lwn.net/ml/linux-fsdevel/20190925225243.GF804@dread.disaster.area/>) early on that corrupt data should just be treated as bad data and this operation could be unprivileged, but that view did not win out. 

Then, encoded data can be read or written using the [`preadv()` and `pwritev()`](<https://man7.org/linux/man-pages/man2/preadv.2.html>) system calls. The new `RWF_ENCODED` flag must be used to indicate that encoded data is being transferred. A normal invocation of these system calls takes an array of pointers to `iovec` structures describing the buffers to be transferred; when encoded I/O is being done, though, the first pointer instead refers to an instance of the new `encoded_iov` structure type: 

```
struct encoded_iov {
    	__aligned_u64 len;
    	__aligned_u64 unencoded_len;
    	__aligned_u64 unencoded_offset;
    	__u32 compression;
    	__u32 encryption;
        };
```

The `len` field must contain the length of this structure; it is there in case new fields are added in the future. The `unencoded_len` and `unencoded_offset` fields describe the portion of the file affected by this operation; the `compression` and `encryption` fields contain filesystem-dependent values describing the type of compression and encryption applied. All other pointers in the `iovec` array point to actual `iovec` structures describing the data to transfer. 

The patch set includes support for reading and writing compressed data from a Btrfs filesystem. There is also [a follow-on patch set](<https://lwn.net/ml/linux-fsdevel/cover.1605723600.git.osandov@fb.com/>) working this support into the `send` and `receive` operations. Benchmarks included there show a significant reduction in bandwidth required to transmit the data, reduced CPU time usage and, in some cases, reduced elapsed time as well. 

This patch series has been through six revisions as of this writing; [the first version](<https://lwn.net/ml/linux-fsdevel/cover.1568875700.git.osandov@fb.com/>) was posted in September 2019. Various implementation issues have been addressed, and the work appears to be converging on something that should be ready to merge soon.
