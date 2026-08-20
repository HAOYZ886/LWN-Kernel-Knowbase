---
title: The new timerfd() API
url: https://lwn.net/Articles/251413/
date: "September 25, 2007"
category: "User-space API; timerfd"
author: "By Jonathan Corbet September 25, 2007"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
September 25, 2007

The `timerfd()` system call was added in the 2.6.22 kernel. The core idea behind `timerfd()` \- allowing a process to associate a file descriptor with timer events - is not controversial, but the implementation of this idea did, belatedly, [raise a few eyebrows](<http://lwn.net/Articles/245533/>). In particular, Michael Kerrisk pointed out that `timerfd()` was inconsistent with (and less powerful than) the existing timer-related system calls, and, besides, the 2.6.22 version did not even work as advertised. After a fair amount of discussion, it became clear that the issues with this system call would not be worked out in the 2.6.23 time frame. So the 2.6.23-rc7 prepatch disabled `timerfd()` altogether in an attempt to prevent application developers from using an API which is going to change. 

Prompted by all of this, Davide Libenzi (the creator of the original `timerfd()` system call) has posted [a proposal for a revised `timerfd()` API](<http://lwn.net/Articles/251227/>). The single system call has turned into three different calls with a few new features. 

Under the new API, an application wanting to create a file descriptor for timer events would make a call to: 

```
int timerfd_create(int clockid);
```

Where `clockid` describes which clock should be used; it will be either `CLOCK_MONOTONIC` or `CLOCK_REALTIME`. The return value will, if all goes well, be the requested file descriptor. 

A timer event can be requested with: 

```
int timerfd_settime(int fd, int flags, const struct itimerspec *timer,
    			struct itimerspec *previous);
```

Here, `fd` is a file descriptor obtained from `timerfd_create()`, and `timer` gives the desired expiration time (and re-arming interval value, if desired). This time is normally a relative time, but if the timer sets the `TFD_TIMER_ABSTIME` bit in `flags`, it will be interpreted as an absolute time instead. If `previous` is not `NULL`, the pointed-to structure will be filled with the previous value of the timer. This ability to obtain the previous value is one of the features which was lacking in the original `timerfd()` implementation. 

That implementation also had no way for an application to simply ask what the current value of the timer was. The new API provides a function for querying a timer non-destructively: 

```
int timerfd_gettime(int fd, struct itimerspec *timer);
```

This system call will store the current expiration time (if any) associated with `fd` into `timer`. 

The `read()` interface is essentially unchanged. A process which reads on a timer file descriptor will block if the timer has not yet expired. It will then read a 64-bit integer value indicating how many times the timer has expired since it was last read. A timer file descriptor can be passed to `poll()`, allowing timers to be handled in an applications main event loop. 

Responses to the new API proposal have been muted at best; hopefully this silence means that developers are happy with the new system calls. The alternative is that this iteration of `timerfd()` will not be reviewed any more extensively than its predecessor was. As things stand, the new set of system calls looks likely to be merged for 2.6.24.
