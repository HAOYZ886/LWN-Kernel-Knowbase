---
title: Proper handling of unknown flags in system calls
url: https://lwn.net/Articles/588444/
date: "February 26, 2014"
category: "Development model-Patterns; System calls"
author: "February 26, 2014 This article was contributed by Michael Kerrisk ."
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

February 26, 2014

This article was contributed by [Michael Kerrisk](<http://man7.org/>).

As noted by various commenters on our earlier article, "[Flags as a system call API design pattern](<https://lwn.net/Articles/585415/>)", there is an important step that is required when equipping a system call with a `flags` argument. If the kernel does not take care to return an error if unknown flags are set, it is setting the stage for a number of compatibility problems in the future. Unfortunately, the history of Linux (and Unix) system call development shows that this lesson has been hard to learn. 

In particular, a system call flags argument (or indeed any input structure argument that has a bit-flags field) should always include a check of the following form in its implementation: 

```
if (flags & ~(FL_XXX | FL_YYY))
    	    return -EINVAL;
```

Here, `FL_XXX` and `FL_YYY` form the hypothetical set of flags that the system call understands, and the effect of this check is to deliver an error when the caller specifies any bit value other than one in the set. Checks like this future-proof the API against the day when the system call understands additional flags. Suppose that the system call adds a new flag, `FL_ZZZ`, and adjusts its check to: 

```
if (flags & ~(FL_XXX | FL_YYY | FL_ZZZ))
    	    return -EINVAL;
```

A user-space application is now able to check whether it is running on a kernel where the system call supports `FL_ZZZ` by checking for an `EINVAL` error when making the system call. This allows the application to flexibly deal with system call differences across kernel versions. 

Although implementing `flags` checks such as the above inside the kernel might seem simple and obvious, it turns out that dozens of system calls _don't_ make this check, including `clock_nanosleep()`, `clone()`, `epoll_ctl()`, `fcntl(F_SETFL)`, `mmap()`, `msgrcv()`, `msgsnd()`, `open()`, `recv()`, `send()`, `sigaction()`, `splice()`, `unshare()`, and many others. 

Most of those system calls have been around for several years. In more recent times, most new system calls that have a `flags` argument include the required check. However, such checks are missing even in a few system calls added in recent kernel versions, such as as `open_by_handle_at()` (2.6.39), `recvmmsg()` (2.6.33), and `sendmmsg()` (3.0). In each of those recent cases, the implementer was presumably emulating the lack of checking that was done in the corresponding earlier system call (`open()`, `recv()`, `send()`). However, the failure to add the checks represents a missed opportunity to improve on the original API. 

For each of the system calls that lack a check on the `flags` argument, user-space applications have no easy way of detecting what API flags a particular kernel version supports. Furthermore, failure to implement such checks in the kernel can also complicate the lives of kernel developers, as a couple of examples demonstrate. 

When the kernel fails to check that only valid bits are passed in `flags`, user-space applications can, with impunity, place random garbage in the "unused" bits of `flags`. If a kernel developer then decides to make use of one of the hitherto unused bits, this may lead to surprising breakage in user-space applications, which in turn may require the kernel developer to write suboptimal implementations of new user-space API features. [One recent example](<https://lwn.net/Articles/520198/#EPOLLWAKEUP>) of this was in the implementation of the `EPOLLWAKEUP` flag, where avoiding user-space breakage meant that the kernel silently ignored this flag if the caller did not have the `CAP_BLOCK_SUSPEND` capability. Ideally, of course, the kernel would have informed the caller by returning an error from the call. Consequently, applications that want to be absolutely sure that the call will succeed must explicitly check beforehand that they have the `CAP_BLOCK_SUSPEND` capability. 

An even more recent example was in the implementation of the `O_TMPFILE` flag for `open()`, where the flag definition [incorporated the `O_DIRECTORY` flag](<https://lwn.net/Articles/558940/>), with the goal that older kernels that do not support `O_TMPFILE` would give an error if the flag was specified in a call to `open()`. This was necessary, because applications that create temporary files are often security conscious, and need to know whether their requests to create hidden temporary files have been honored. Without this fix, the `O_TMPFILE` flag would be silently ignored on older kernels, and [an application might end up creating a visible file](<http://thread.gmane.org/gmane.linux.file-systems/75969/focus=76261>). An unpleasant side effect of that implementation is that user-space applications [must check for two different errors](<http://man7.org/linux/man-pages/man2/open.2.html#BUGS>) from `open()` in order to determine whether they are running on a kernel that doesn't support `O_TMPFILE`. 

Finally, it is worth mentioning that a few system calls have added the required `flags` check _after_ the call was first implemented. Two examples are the ancient system calls `umount2()` (check added in Linux 2.6.34) and `swapon()` (check added in Linux 3.4). In addition, the `mremap()` call, which first appeared in Linux 2.0, added the check in Linux 2.4, and the `timerfd_settime()` system call, which first appeared in Linux 2.6.25, added the check in Linux 2.6.29. 

However, the addition of `flags` checks to these system calls represents an exception to the general rule that such checks cannot be added after the fact, because doing so would break existing applications that happen to pass random garbage in the "unused" bits of `flags`. With `umount2()` and `swapon()`, the change was possible presumably because there are few users of these system calls other than the `mount` and `swapon` commands, and those programs could be modified if the kernel change caused them to break. In the case of `timerfd_settime()`, the change was made soon after the initial implementation, when there were likely to have been few users of the interface. And in the case of `mremap()`, the change was made at the time of a major kernel version change (from 2.2 to 2.4), when such ABI changes were occasionally permitted; with the contemporary 10-week release cycle, such changes are not permitted. 

Thus if the check on unused flag bits is not included in the initial implementation, it is often impossible to add it later. The clear conclusion is that any addition of flag bits to a system call should come with the proper checks from the outset.
