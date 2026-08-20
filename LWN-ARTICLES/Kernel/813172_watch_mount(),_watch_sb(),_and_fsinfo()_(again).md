---
title: "watch_mount(), watch_sb(), and fsinfo() (again)"
url: https://lwn.net/Articles/813172/
date: "February 24, 2020"
category: System calls
author: "By Jonathan Corbet February 24, 2020"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 24, 2020

Filesystems, by design, hide a lot of complexity from users. At times, though, those users need to be able to look inside the black box and extract information about what is going on within a filesystem. Answering this need is David Howells, the creator of a number of filesystem-oriented system calls; in [this patch set](<https://lwn.net/ml/linux-kernel/158230810644.2185128.16726948836367716086.stgit@warthog.procyon.org.uk/>) he tries to add three more, one of which we have seen before and two of which are new. 

The new system calls, `watch_mount()` and `watch_sb()`, provide ways for a process to request notifications whenever something changes at either a mount point (`watch_mount()`) or within a specific mounted filesystem (`watch_sb()`, the "`sb`" standing for "superblock"). For a mount point, events of interest can include the mounting or unmounting of filesystems anywhere below the mount point, the change of an attribute like read-only, movement of mount points, and more. Filesystem-specific events can also include attribute changes, along with filesystem errors, quota problems, or network issues for remote filesystems. 

These system calls are built on a newer version of the [event-notification mechanism](<https://lwn.net/Articles/790831/>) that Howells has been working on for some time. In the past, getting notifications has involved opening a new device (`/dev/watch_queue`), but that interface has changed in the meantime. In the current version, a process calls [`pipe2()`](<http://man7.org/linux/man-pages/man2/pipe.2.html>) with the new `O_NOTIFICATION_PIPE` flag to create a special type of pipe meant for notification use. The writable side of this pipe is not used by the application; the file descriptor for the readable end can be passed to either of the new system calls: 

```
int watch_mount(int dirfd, const char *path, unsigned int flags,
        		    int watch_fd, int watch_id);
        int watch_sb(int dirfd, const char *path, unsigned int flags,
        		 int watch_fd, int watch_id);
```

In both cases, `dirfd`, `path`, and `flags` identify the directory of interest in the usual [`openat()`](<http://man7.org/linux/man-pages/man2/open.2.html>) style. The notification pipe is passed in as `watch_fd`, and `watch_id` is an integer value that will be returned in any generated events. There is a special case, though; if `watch_id` is `-1`, any existing watch using the given `watch_fd` will be removed. 

The application receives events by reading from the pipe. By default all events affecting the given watch point will be returned. The application can, though, create a filter that is attached to the notification pipe with an `ioctl()` call. There's another `ioctl()` call to set the size of the buffer used to hold notifications sent to user space. Curious readers can see these system calls used in [this sample program](<https://git.kernel.org/pub/scm/linux/kernel/git/dhowells/linux-fs.git/tree/samples/watch_queue/watch_test.c?h=fsinfo-core>). 

Unlike the system calls described above, `fsinfo()` has been [seen before](<https://lwn.net/Articles/792628/>). Its prototype remains the same: 

```
int fsinfo(int dirfd, const char *path, const struct fsinfo_params *params,
    	       void *buffer, size_t buf_size);
```

As before, `dirfd` and `path` describe the filesystem for which information is requested; there is no `flags` argument here, but it is hidden within the `params` structure, which looks like this: 

```
struct fsinfo_params {
    	__u32	at_flags;
    	__u32	flags;
    	__u32	request;
    	__u32	Nth;
    	__u32	Mth;
    	__u64	__reserved[3];
        };
```

The `at_flags` field contains the same flags that one would ordinarily expect to see in an `openat()`-style system call. The `request` field describes the information that is being asked for; a number of possible values can be found in [this patch](<https://lwn.net/ml/linux-fsdevel/158230816405.2185128.14624101691579582829.stgit@warthog.procyon.org.uk/>) from the series. Potentially available information ("potentially" because filesystems are not required to implement every possibility) include filesystem limits, timestamp resolution information, the volume UUID, the servers behind a remote filesystem, and more. For attributes that can have multiple values, the `Nth` and `Mth` fields can be used to select one in particular. 

The format of the returned value is ... complex. Values are stored into the provided `buffer` in any of a number of formats, depending on what was requested. For some, a structure is returned; others return a string or a type called simply "opaque". There is some documentation in [this patch](<https://lwn.net/ml/linux-fsdevel/158230822028.2185128.5408262159157374165.stgit@warthog.procyon.org.uk/>), but it seems clear that potential users of this system call will have to do some digging to figure out the information that will be returned to them. 

This patch set is now in its 17th revision, having evolved quite a bit over the years. The one comment on this version, so far, comes from James Bottomley, who [suggested](<https://lwn.net/ml/linux-fsdevel/1582316494.3376.45.camel@HansenPartnership.com/>) that there may not be a need for `fsinfo()` at all. Instead, with some changes to how `fsconfig()` (which is used to configure filesystem attributes) is implemented, it could be turned into an interface that could both set and read attributes. So far, Howells has not responded to that suggestion. 

Overall, the fact that these patches have been through 17 revisions (so far) says a lot. Nobody doubts that getting this information out of the kernel would be useful, but the API remains complex and hard for potential users to understand. Whether that can be fixed while retaining the features provided by these system calls is not clear, though.
