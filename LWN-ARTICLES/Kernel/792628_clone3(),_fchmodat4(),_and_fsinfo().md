---
title: "clone3(), fchmodat4(), and fsinfo()"
url: https://lwn.net/Articles/792628/
date: "July 5, 2019"
category: System calls
author: "By Jonathan Corbet July 5, 2019"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 5, 2019

The kernel development community continues to propose new system calls at a high rate. Three ideas that are currently in circulation on the mailing lists are `clone3()`, `fchmodat4()`, and `fsinfo()`. In some cases, developers are just trying to make more flag bits available, but there is also some significant new functionality being discussed. 

#### `clone3()`

The `[clone()](<http://man7.org/linux/man-pages/man2/clone.2.html>)` system call creates a new process or thread; it is the actual machinery behind [`fork()`](<http://man7.org/linux/man-pages/man2/fork.2.html>). Unlike `fork()`, `clone()` accepts a `flags` argument to modify how it operates. Over time, quite a few flags have been added; most of these control what resources and namespaces are to be shared with the new child process. In fact, so many flags have been added that, when `CLONE_PIDFD` was merged for 5.2, the last available flag bit was taken. That puts an end to the extensibility of `clone()`. 

The natural solution is to clone the `clone()` system call into a new one that would be able to accept more flags. Christian Brauner, perhaps feeling guilty for having snagged the last flag for `CLONE_PIDFD`, set out to do this work. His first attempt was called [`clone6()`](<https://lwn.net/ml/linux-kernel/20190526102612.6970-1-christian@brauner.io/>) but, after some discussion, it was downgraded to [`clone3()`](<https://lwn.net/ml/linux-kernel/20190604160944.4058-1-christian@brauner.io/>). (For the curious, there is a `clone2()` that appears to only be of interest on the ia64 architecture). The prototype for this system call looks something like this: 

```
struct clone_args {
            u64 flags;
            int *pidfd;
            int *child_tid;
            int *parent_tid;
            int exit_signal;
            unsigned long stack;
            unsigned long stack_size;
            unsigned long tls;
        };
    
        int clone3(struct clone_args *args, size_t size);
```

The `clone_args` structure contains much of the information that was previously passed directly to `clone()` or crammed into the `flags` field. The new `flags` is wider (64 bits on all architectures) and regains some space due to the relocation of information like the exit signal number. That should provide enough flags to last, as they say, "for a while". 

The `size` argument is the size of the `clone_args` structure itself. Should there ever be a need to expand that structure in the future, the kernel will be able to tell whether any given user-space caller is using the new or the old version of the structure by examining `size` and do the right thing either way. So, with luck, there should be no need to create a `clone4()` anytime soon. 

This interface seems to be satisfactory to everybody involved, though Jann Horn did [point out](<https://lwn.net/ml/linux-kernel/CAG48ez2wyDhM-V1hs5ya1R4x7wHT=T8XLOYCPUyw97kzzLhbhg@mail.gmail.com/>) one significant problem: the seccomp mechanism is unable to examine system-call arguments that are passed in separate structures, so it will be unable to make decisions based on the flags given to `clone3()`. That, he said, means that code meant to be sandboxed with seccomp may not use `clone3()` at all. Kees Cook has [suggested](<https://lwn.net/ml/linux-kernel/201905301122.88FD40B3@keescook/>) a new mechanism for fetching user-space data for system calls that could be used by seccomp, but nobody appears to be working on that idea currently. 

Meanwhile, `clone3()` is in linux-next, and so can be expected to appear in 5.3. 

#### `fchmodat4()`

A look at [the man page for `fchmodat()`](<http://man7.org/linux/man-pages/man2/chmod.2.html>) reveals the following prototype: 

```
int fchmodat(int dirfd, const char *pathname, mode_t mode, int flags);
```

The `flags` argument is documented to have one possible value: `AT_SYMLINK_NOFOLLOW`, which would cause `fchmodat()` to operate directly on a symbolic link rather than its target. There's only one little problem: `fchmodat()` as implemented in the kernel does not actually accept a `flags` argument. That is why the man page concludes with: ""This flag is not currently implemented"". 

Palmer Dabbelt was motivated to action by a seemingly unpleasant experience: ""I spent half of dinner last night being complained to by one of our hardware engineers about Linux's lack of support for the flags argument to `fchmodat()`"". The result was [a patch set](<https://lwn.net/ml/linux-kernel/20190531191204.4044-1-palmer@sifive.com/>) implementing support for `fchmodat4()`, which has the same prototype as `fchmodat()` but which actually implements the `flags` argument. 

This patch set seems uncontroversial, so there should be no real barrier to its merging, though it has not yet found its way into linux-next. 

#### `fsinfo()`

The [`statfs()`](<http://man7.org/linux/man-pages/man2/statfs.2.html>) system call can be used to get certain types of information about a filesystem, including its format, block size, available free blocks, maximum file-name length, and so on. But it turns out that there is a lot more to know about a filesystem than that, and `statfs()` is unable to provide that information. It seems like a situation just begging for somebody to come along and implement `statfs2()`, but instead we get [`fsinfo()`](<https://lwn.net/ml/linux-kernel/156173661696.14042.17822154531324224780.stgit@warthog.procyon.org.uk/>) from David Howells. 

The prototype for `fsinfo()` looks like this: 

```
struct fsinfo_params {
    	__u32	at_flags;
    	__u32	request;
    	__u32	Nth;
    	__u32	Mth;
    	__u64	__reserved[3];
        };
    
        int fsinfo(int dfd, const char *filename,
        	       const struct fsinfo_params *params, void *buffer,
    	       size_t buf_size);
```

The `dfd` and `filename` arguments identify the filesystem about which information is needed. `params` is an optional array describing the requested information, while `buffer` and `buf_size` define the output buffer. 

If `params` is null, the returned information will be essentially the same as what `statfs()` would provide. But it is possible to get more, including limits on the filesystem's capabilities, timestamp resolution, mount-time parameters, remote server information, and more. Once [this patch set](<https://lwn.net/ml/linux-kernel/156173681842.14728.9331700785061885270.stgit@warthog.procyon.org.uk/>) is applied, `fsinfo()` will also be able to return information about the system's mount topology. 

This system call is complex, to say the least; there is not space here to try to describe how it all works. Fortunately, there is some good documentation provided with it. [This patch](<https://lwn.net/ml/linux-kernel/156173671169.14042.2848093993797220779.stgit@warthog.procyon.org.uk/>) provides a fair amount of information about what `fsinfo()` can do, liberally intermixed with API information for filesystem developers. But see also [this patch](<https://lwn.net/ml/linux-kernel/156173687101.14728.6401356872306559415.stgit@warthog.procyon.org.uk/>) for information on how the mount-topology queries work, and [this one](<https://lwn.net/ml/linux-kernel/156173668587.14042.3095176129086856318.stgit@warthog.procyon.org.uk/>) for the somewhat baroque mechanism used to format parameter values passed back to user space. 

While there is clear value in the creation of an interface for extracting arbitrary filesystem-related information from the kernel, the complexity of the `fsinfo()` patch set has proved daunting to reviewers, who have asked for it to be broken up in the past. Filesystem developers have, in recent years, become more insistent that new features come with additions to the xfstests suite as well; those have not yet been provided in this case. `fsinfo()` has been circulating for a while — Howells [posted a version](<https://lwn.net/ml/linux-api/153314002975.18964.1773855756473041897.stgit@warthog.procyon.org.uk/>) nearly one year ago — but chances are good that it will need to circulate for a bit longer still before it's ready for the mainline.
