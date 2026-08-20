---
title: quotactl_path() becomes quotactl_fd()
url: https://lwn.net/Articles/859679/
date: "June 14, 2021"
category: "Releases-5.13; System calls-quotactl fd"
author: "By Jonathan Corbet June 14, 2021"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
June 14, 2021

The [`quotactl()` system call](<https://man7.org/linux/man-pages/man2/quotactl.2.html>) is used to manipulate disk quotas on a filesystem; it can be used to turn quota enforcement on or off, change quotas, retrieve current usage information, and more. The 5.13 merge window brought in a new variant of that system call that was subsequently disabled due to API concerns; its replacement is now taking form. 

`quotactl()` has the following prototype: 

```
int quotactl(int cmd, const char *special, int id, caddr_t addr);
```

Quota manipulation is a filesystem-oriented operation, but `quotactl()` requires that the caller provide the name of the block device containing the target filesystem (as `special`). That can require users (who typically think in terms of mount points) or user-space tools to do a lookup to find out which block device underlies the filesystem of interest. For filesystems that do not _have_ an underlying block device, `quotactl()` is not usable at all; among others, the [UBIFS](<https://lwn.net/Articles/276025/>) filesystem, which uses persistent memory for storage, is affected by this problem. 

The new system call was introduced to address these issues: 

```
int quotactl_path(unsigned int cmd, const char *mountpoint, qid_t id,
        		      void *addr);
```

Rather than accepting a device name, `quotactl_path()` uses the path name of the mount point where the filesystem is to be found. 

Shortly after this new system call was pulled into the mainline (during the 5.13 merge window), an LWN reader [pointed out](<https://lwn.net/Articles/855954/>) that `quotactl_path()` failed to follow contemporary best practices for system calls. In particular, it lacked a "flags" argument for future extensions, and there was no directory file-descriptor argument as found in system calls like [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>). In response to those criticisms, the new system call was [disabled](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=5b9fedb31e47>) in the 5.13-rc3 release so that the problems could be fixed. 

On June 2, Jan Kara posted [a proposed replacement](<https://lwn.net/ml/linux-fsdevel/20210602151553.30090-1-jack@suse.cz/>) for `quotactl_path()`. Rather than simply add the new parameters, though, the new call does away with the path name entirely: 

```
int quotactl_fd(unsigned int fd, unsigned int cmd, qid_t id, void *addr);
```

With `quotactl_fd()`, the caller is expected to provide an open file descriptor identifying the filesystem to operate on. The advantage of doing things this way is that the full complexity of handling the directory file descriptor, resolve flags, etc. can be deferred to `openat2()` rather than being reimplemented here. For usage involving multiple `quotactl_fd()` calls, reusing a file descriptor that only need be opened once should be more efficient anyway. 

Note that there still is no "flags" argument. In this case, that should not be problematic; the system call can always be extended by adding new commands. In short, as Kara pointed out in the cover letter, the new system call looks a lot like [`ioctl()`](<https://man7.org/linux/man-pages/man2/ioctl.2.html>), with all the flexibility associated with that call. 

Looked at from one angle, this incident could appear to be a failure of the process; how did a system call with these problems get into the mainline? Sometimes code is rushed into the kernel without adequate review, but this patch set was posted three times (in [January](<https://lwn.net/ml/linux-kernel/20210128141713.25223-1-s.hauer%40pengutronix.de/>), [February](<https://lwn.net/ml/linux-kernel/20210211153024.32502-1-s.hauer@pengutronix.de/>), and [March](<https://lwn.net/ml/linux-kernel/20210304123541.30749-1-s.hauer@pengutronix.de/>)) before being merged. Developers clearly looked at the patches and a number of changes were made, but nobody caught the API problems. 

But this episode could also be seen as an example of the process working exactly as it is supposed to. The period between the merge window and the 5.13 release is there to catch remaining issues, and this one was duly pointed out. The system call was immediately disabled to ensure that it would not be released in that form, after which it would likely have been necessary to maintain it indefinitely. And now a better replacement has been created. 

That replacement has been reviewed (and approved) by a few relevant developers, including Sascha Hauer, the author of the original `quotactl_path()` implementation. It has not, as of 5.13-rc6, been pushed into the mainline kernel. Kara hasn't said whether he plans to get this work into 5.13 before its release; if so, the time to do so is quickly running out. The alternative is to push it during the 5.14 merge window; that would allow for another cycle of review to be sure that there are truly no more lingering issues. Given that this API has needed one rework and that 5.13 is due in less than two weeks, waiting may well prove to be the more prudent approach.
