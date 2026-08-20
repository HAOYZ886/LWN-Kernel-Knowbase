---
title: The return of SEEK_HOLE
url: https://lwn.net/Articles/440255/
date: "April 26, 2011"
category: FIEMAP ioctl; Filesystems
author: "By Jonathan Corbet April 26, 2011"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 26, 2011

Back in 2007, LWN readers learned about the [SEEK_HOLE and SEEK_DATA](<https://lwn.net/Articles/260795/>) options to the `lseek()` system call. These options allow an application to map out the "holes" in a sparsely-allocated file; they were originally implemented in Solaris for the ZFS filesystem. At that time, this extension was rejected for Linux; the Linux filesystem developers thought they had a better way to solve the problem. In the end, though, it may have turned out that the Solaris crew had the better approach. 

Filesystems on POSIX-compliant systems are not required to allocate blocks for files if those blocks would contain nothing but zeros. A range within a file for which blocks have not been allocated is called a "hole." Applications which read from a hole will get lots of zeros in response; most of the time, applications will not be aware that the actual underlying storage has not been allocated. Files with holes are relatively rare, but some applications do create "sparse" files which are more efficiently stored if the holes are left out. 

Most of the time, applications need not care about holes, but there are exceptions. Backup utilities can save storage space if they notice and preserve the holes in files. Simple utilities like `cp` can also, if made aware of holes, ensure that those holes are not filled in any copies made of the relevant files. Thus, it makes sense for the system to provide a way for applications which care to learn about where the holes in a file - if any - may be found. 

The interface created at Sun used the `lseek()` system call, which is normally used to change the read/write position within a file. If the `SEEK_HOLE` option is provided to `lseek()`, the offset will be moved to the beginning of the first hole which starts after the specified position. The `SEEK_DATA` option, instead, moves to the beginning of the first non-hole region which starts after the given position. A "hole," in this case, is defined as a range of zeroes which need not correspond to blocks which have actually been omitted from the file, though in practice it almost certainly will. Filesystems are not required to know about or report holes; `SEEK_HOLE` is an optimization, not a means for producing a 100% accurate map of every range of zeroes in the file. 

When Josef Bacik posted his 2007 `SEEK_HOLE` patch, it was [received](<http://thread.gmane.org/gmane.linux.file-systems/19622/focus=19624>) with comments like: 

I stand by my belief that SEEK_HOLE/SEEK_DATA is a lousy interface. It abuses the seek operation to become a query operation, it requires a total number of system calls proportional to the number holes+data and it isn't general enough for other similar uses (e.g. total number of contiguous extents, compressed extents, offline extents, extents currently shared with other inodes, extents embedded in the inode (tails), etc.) 

So this patch was not merged. What we got instead was a new `ioctl()` operation called `FIEMAP`. There can be no doubt that `FIEMAP` is a more powerful operation; it allows the precise mapping of the extents in the file, with knowledge of details like extents which have been allocated but not written to and those which have been written to but which do not, yet, have exact block numbers assigned. Information for multiple extents can be had with a single system call. With an interface like this, it was figured, there is no need for something like `SEEK_HOLE`. 

Recently, though, Josef has posted [a new `SEEK_HOLE` patch](<https://lwn.net/Articles/439719/>) with the comment: 

Turns out using fiemap in things like cp cause more problems than it solves, so lets try and give userspace an interface that doesn't suck. 

A quick search on the net will turn up a long list of bug reports related to `FIEMAP`. Some of them are simply bugs in specific filesystem implementations, like [the problems related to delayed allocation](<https://lwn.net/Articles/429345/>) that were discovered in February. Others have to do with the rather complicated semantics of some of the `FIEMAP` options and whether, for example, the file in question must be synced to the disk before the operation can be run. And others just seem to be related to the complexity of the system call itself. The end result has been a long series of reports of corrupted files - not the sort of thing filesystem developers want to find in their mailboxes. 

It seems that `FIEMAP` is a power tool with sharp edges which has been given to applications which just wanted a butter knife. For the purpose of simply finding out which parts of a file need not be copied, a simple interface like `SEEK_HOLE` seems to be more appropriate. So, one assumes, this time the interface will likely get into the kernel. 

That said, it looks like a few tweaks will be needed first. The API as posted by Josef does not exactly match what Solaris does; to add an API which is not compatible with the existing Solaris implementation makes little sense. There is also the question of what happens when the underlying filesystem does not implement the `SEEK_HOLE` and `SEEK_DATA` options; the current patch returns `EINVAL` in this situation. A proposed alternative is to have a VFS-level implementation which just assumes that the file has no holes; that makes the API appear to be supported on all filesystems and eliminates one error case from applications. 

Once these details are worked out - and appropriate man pages written - `SEEK_HOLE` should be set to be merged this time around. `FIEMAP` will still exist for applications which need to know more about how files are laid out on disk; tools which try to optimize readahead at bootstrap time are one example of such an application. For everything else, though, there should be - finally - a simpler alternative.
