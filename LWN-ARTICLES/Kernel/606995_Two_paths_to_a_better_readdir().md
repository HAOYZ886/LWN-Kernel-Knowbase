---
title: Two paths to a better readdir()
url: https://lwn.net/Articles/606995/
date: "July 30, 2014"
category: Filesystems; dirreadahead; xgetdents
author: "By Jonathan Corbet July 30, 2014"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
July 30, 2014

A common filesystem workload follows a simple pattern: work through a list of files in a directory, and use `stat()` to obtain information about each of those files. The "`ls -l`" command is a classic example of this type of workload, but there are many others. This workload has always run more slowly on Linux systems than developers would like, but getting a solution into the kernel has happened even more slowly. Recently, a pair of possible solutions was posted by Abhi Das; perhaps this time this issue will be resolved — in a surprising way. 

The problem with the "`ls -l`" workload is simple enough: two system calls are required for each file of interest. A call to `getdents()` (usually via the `readdir()` function in the C library) obtains the name of a file in the directory; then `stat()` is called to get the information about that file. The `stat()` call, in particular, can be expensive, with each call forcing the underlying filesystem to perform I/O to obtain the desired information. In some cases, that information may be spread across multiple on-disk data structures, requiring even more I/O, even if the calling application does not actually use everything that `stat()` returns. Doing all this work is inefficient; it would be nice if there were a way to limit the information gathered to what the application actually needs and to get that information in batches. 

This issue is not new; it was, in fact, already somewhat old when it was discussed at the [2009 Linux Storage and Filesystem Workshop](<https://lwn.net/Articles/327740/#readdirplus>). A proposed solution, in the form of the [`xstat()` system call](<https://lwn.net/Articles/394298/>), was posted in 2010 but did not get very far. At this point, well into 2014, some filesystems have code to try to optimize for this kind of workload, but there is still no general solution in the kernel. For the last few years, there has appeared to be little interest among developers in working on this problem. 

In that setting, Abhi has come forward with two independent solutions demonstrating two separate approaches to the problem. His hope is to get feedback on both and, once one of them emerges as the preferred solution, get it into the mainline kernel. 

#### xgetdents()

The [first approach](<https://lwn.net/Articles/606713/>) builds on the 2010 `xstat()` system call by David Howells. It adds two new system calls: 

```
int xstat(int dirfd, const char *filename, unsigned int flags,
        	      unsigned int mask, struct xstat *info);
        int fxstat(int fd, unsigned int flags, unsigned int mask, struct xstat *info);
```

The first form looks up a given file by name, while the second returns information for an open file identified by its descriptor. The `flags` field is there to change the operation of the system call; there is little use of it in this patch set. Of more interest is `mask`, which tells the kernel which information is being requested by the application. There are quite a few bits that can be set here; examples include `XSTAT_MODE` (for the file protection bits), `XSTAT_UID` (file owner), `XSTAT_RDEV` (underlying storage device), `XSTAT_ATIME` (last access time), or `XSTAT_INO` (inode number). `XSTAT_ALL_STATS` can be used to request all available information. On a successful return, the `info` structure will be filled in with the requested data. 

On top of this work, Abhi has added another system call: 

```
int xgetdents(unsigned int fd, unsigned int flags, unsigned int mask,
    		  void *buf, unsigned int count);
```

Here, `fd` is a file descriptor for the directory of interest, while `flags` and `mask` are as above (though `mask` has been extended to allow the application to request various types of extended attribute data). Information is returned in `buf`, which is a simple byte array, `count` bytes in length. The `xgetdents()` call will attempt to retrieve information about multiple files in the given directory until `buf` fills. 

The actual data returned in `buf` is somewhat complex. The top-level structures defining this information are: 

```
struct xdirent_blob {
    	unsigned int    xb_xattr_count;
    	char            xb_blob[1]; /* contains variable length data like
    				     * NULL-terminated name, xattrs etc */
        };
    
        struct linux_xdirent {
    	unsigned long        xd_ino;
    	char                 xd_type;
    	unsigned long        xd_off;
    	struct xstat         xd_stat;
    	unsigned long        xd_reclen;
    	struct xdirent_blob  xd_blob;
        };
```

The documentation of the return format is somewhat sparse. Actually, it does not exist at all, so one is forced to reverse-engineer it from the code. It appears that information for each file will be returned in one variable-length `linux_xdirent` structure. The name of the file is the first thing stored in `xd_blob`, followed by extended attribute information if that has been requested. This structure clearly requires a bit of work to understand and pick apart on the user-space side, but it does have the advantage of allowing all of that information to be collected and returned in a single system call. 

#### dirreadahead()

The [alternative approach](<https://lwn.net/Articles/606712/>) is rather simpler. It adds a single system call: 

```
int dirreadahead(unsigned int fd, loff_t *offset, unsigned int count);
```

This call will attempt to initiate the reading of file information for `count` files in the directory represented by `fd`, starting at the given `offset` within the directory. The `offset` value will be updated to reflect the number of files whose information was actually read. One can thus use multiple `dirreadahead()` calls to work through a directory with the kernel maintaining the `offset` value as things progress. 

In this case, it is still necessary to call `getdents()` and `stat()` to get the needed information. What changes is that, with luck, the filesystem will have already pulled that information into an internal cache, so the calls should be handled quickly. Reading information for multiple files at once allows batching to be done; even if the information is dispersed on physical media, the necessary I/O operations can be reordered for optimal execution. 

The [introductory message](<https://lwn.net/Articles/607019/>) to the two patch sets included some benchmark results on the GFS2 filesystem. Both approaches performed better than mainline kernels when presented with a workload heavy with `getdents()` and `stat()` system calls. Perhaps surprisingly, `dirreadahead()` consistently performed far better than `xgetdents()`. That result may be an artifact of the `xgetdents()` implementation or of the GFS2 filesystem, but it shows that the far simpler readahead-based approach is worthy of consideration. 

The readahead idea quickly led to questions of whether the kernel could somehow perform this readahead automatically, as it does with basic file I/O. Trond Myklebust [noted](<https://lwn.net/Articles/607022/>) that the NFS client tries to detect workloads where this kind of readahead might be of value. In the general case, though, this detection is hard to do; there is no obvious connection within the kernel between the `getdents()` and `stat()` calls. So, for now at least, it may be up to user space to communicate that information directly. Either of the two interfaces described here could be used for that communication, but it seems that the relative simplicity of the `dirreadahead()` approach would argue strongly in its favor, even in the absence of better benchmark results.
