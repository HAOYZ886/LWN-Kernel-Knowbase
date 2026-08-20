---
title: "System call updates: indirect(), timerfd(), and hijack()"
url: https://lwn.net/Articles/260172/
date: "November 28, 2007"
category: Containers; System calls; indirect; timerfd
author: "By Jonathan Corbet November 28, 2007"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
November 28, 2007

[Last week's discussion](<http://lwn.net/Articles/259068/>) of the proposed `indirect()` system call ended with some complaints from developers on the ugliness of the interface. Since then there has been some talk about system call interfaces in general, but not a whole lot of ideas for how `indirect()` could be done better. 

The leading alternative would be that pushed by H. Peter Anvin: rather than use `indirect()` to extend a system call, simply make a new system call with the desired additional parameters. Then, usually, the old implementation can be replaced with a simple stub which calls the new version with the default values for the new parameters. It is a simple approach which easily maintains binary compatibility with very little runtime cost. Since there is no particular shortage of system call numbers, this is a process which could go on for a long time. 

The management of increasing numbers of system calls does impose a cost, though; each one of those system calls is a user-space API which cannot ever be broken. The `indirect()` approach, instead, does not add more system calls. As long as the addition of parameters (with default values of zero) is done with care, avoiding API problems should be relatively easy to do. 

There are also limits on how many parameters can be easily passed to system calls; on most systems, that limit is around six. Any system call requiring more arguments must already do uncomfortable things with indirect blocks. Creating new system calls with additional parameters will create more cases where this sort of indirect parameter handling is required. So the approach used by `indirect()` will find itself being used, in some form, anyway. 

The key argument, though, still appears to be the syslet/threadlet mechanism. The ability to make any system call asynchronous has a lot of appeal, but doing so requires some additional information - a place to store the result of the call, if nothing else. Asynchronous system calls, in Linux, are, for all practical purposes, a type of indirect call. The proposed `indirect()` interface looks like it should be able to accommodate asynchronous calls nicely - though the precise API has not, yet, been nailed down. 

As a result of all this, chances are that some form of `indirect()` will find its way into the mainline - though there is still time for somebody to come up with a better idea. 

Meanwhile, the [last time](<https://lwn.net/Articles/251413/>) `timerfd()` was discussed here, it had been disabled in the 2.6.23 kernel as a result of complaints about its interface. Since then, little has happened with `timerfd()`, with the result that it will almost certainly not be present in 2.6.24 either. Some work has been done with this system call, though, and [a new API proposal](<http://lwn.net/Articles/259777/>) has been posted. This version has three system calls, the first of which is `timerfd_create()`: 

```
int timerfd_create(int clockid, int flags);
```

The `clockid` argument tells the system which clock should be used: `CLOCK_MONOTONIC` or `CLOCK_REALTIME`. The `flags` argument is a recent addition; it is currently unused and must be zero. It was added on the assumption that somebody, somewhere, will always want some sort of behavior modification and one might as well avoid the need for an indirect version while it's easy. The return value from `timerfd_create()` is a file descriptor which can be passed to `read()` or any of the `poll()` variants. But, first, the timer should probably be programmed with: 

```
int timerfd_settime(int fd, 
                            int flags,
    		        const struct itimerspec *timer,
    		    	struct itimerspec *old_timer);
```

Here, `fd` is a file descriptor obtained from `timerfd_create()`, `flags` contains `TFD_TIMER_ABSTIME` if the timer is being set to an absolute time, and `timer` is the expiration time for the timer. If `old_timer` is not `NULL`, the location pointed to will be set to the previous value of the timer. 

It is also possible to query the value of the timer with: 

```
int timerfd_gettime(int fd, struct itimerspec *timer);
```

The value returned in `*timer` will be the current setting of the timer associated with `fd`. 

There's not been a whole lot of comments on this version of the API, so something very similar to it will probably be merged. It would normally be considered to be too late to put a change like this into 2.6.24, but the [2.6.24-rc3-mm2](<http://lwn.net/Articles/260127/>) patch log says "Probably 2.6.24?". So one never knows. If this change is not merged soon, it will almost certainly become available for 2.6.25. 

Finally, the `hijack()` system call continues to be developed on relatively quiet kernel subsystem lists. This call ([described here in October](<http://lwn.net/Articles/256389/>)) behaves much like `clone()` in that it creates a new process. Unlike `clone()`, however, `hijack()` causes the new process to share resources with a specified third process rather than with the parent. Its main reason for existence is to make it easy to enter different namespaces. 

The `hijack()` interface remains almost unchanged: 

```
int hijack(unsigned long clone_flags, int which, int id);
```

The specified `id` value is interpreted according to `which`, which now has three possible values: 

  * `HIJACK_PID` says that `id` is a process ID; the newly-created process will share resources (including namespaces) with the indicated process. 

  * `HIJACK_CG` says that `id` is an open file descriptor for the `tasks` file in a target control group. In this case, the kernel will find a process within that control group and use it as the source for resources and namespaces. 

  * `HIJACK_NS` is the newest option; like `HIJACK_CG`, it is an open file descriptor indicating a control group. In this case, though, only the control group itself and any associated namespaces will be inherited by the new process. This version is intended for use when entry into an empty control group (where there are no processes to inherit from) is desired. 

This new system call still has not seen any exposure on linux-kernel; it may well not survive its first experience there in its current form. If nothing else, a name change (to something which is more descriptive of the real function and, preferably, which does not put users onto intelligence agency watch lists) may well be called for. But a full container implementation on Linux will clearly need some sort of `enter_container()` system call at some point.
