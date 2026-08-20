---
title: timerfd() and system call review
url: https://lwn.net/Articles/245533/
date: "August 14, 2007"
category: "Development model-User-space ABI; User-space API; timerfd"
author: "By Jonathan Corbet August 14, 2007"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
August 14, 2007

One of the fundamental principles of Linux kernel development is that user-space interfaces are set in stone. Once an API has been made available to user space, it must, for all practical purposes, be supported (without breaking applications) indefinitely. There have been times when this rule has been broken, but, even in the areas known for trouble (sysfs, for example), the number of times that the user-space API has been broken has remained relatively small. 

Now consider the `timerfd()` system call, which was added to the 2.6.22 kernel. The purpose of this call is to allow an application to obtain a file descriptor to use with timer events, eliminating the need to use signals. The system call prototype, as found in 2.6.22, is: 

```
long timerfd(int fd, int clockid, int flags, struct itimerspec *utimer);
```

If `fd` is `-1`, a new timer file descriptor will be created and returned to the application. Otherwise, a timer will be set using the given `clockid` for the time specified in `utimer`. The `TFD_TIMER_ABSTIME` flag can be set to indicate that an absolute timer expiration is needed; otherwise the specified time is relative to the current time. The `flags` argument can also be used to request a repeating timer. 

There is another aspect to the `timerfd()` API, though: a read on the timer file descriptor will return an integer value saying how many times the timer has fired since the previous read. If no timer expirations have happened, the `read()` call will block. In the 2.6.22 kernel, the returned value was 32 bits (on all architectures). It has since been decided that a 64-bit value would have been more appropriate, and a patch making that change has been merged for 2.6.23. The 2.6.22.2 stable update also contained the API change. 

That is not the full story, though. Michael Kerrisk, while writing manual pages for the new system call, [encountered a couple of other shortcomings](<https://lwn.net/Articles/245544/>) with the interface. In particular, it is not possible to ask the system for the amount of time remaining on a timer. Other timer-related system calls allow for this sort of query, either as a separate operation or when changing a timer. Michael thought that the `timerfd()` system call should work similarly to those which came before. 

Michael has now [posted a patch](<http://lwn.net/Articles/245373/>) fixing up the `timerfd()` interface. With this patch, the system call would now look like this: 

```
long timerfd(int fd, int clockid, int flags, struct itimerspec *utimer,
                         struct itimerspec *outmr);
```

The new `outmr` pointer must be `NULL` when the file descriptor is first being created. In any other context, it will be used to return the amount of time remaining at any `timerfd()` call. So user space can query a timer non-destructively by calling `timerfd()` with a `NULL` value for `utimer`. If both timer pointers are non-`NULL`, the timer will be set to `utimer`, with its previous value being returned in `outmr`. 

This is, of course, an entirely incompatible change to an API which has already been exported to user space; any code which is using `timerfd()` now will break if it is merged. By the rules, such a change should not be merged, but it appears that there is a good chance that the rules will be bent this time around. One can argue that, in a real sense, the API has not yet been made available to user space: there has been no glibc release which supports `timerfd()`. The number of applications using this system call must be quite low - if, in fact, there are any at all. So a change at this point, especially if it can get into 2.6.23, will improve the interface without actually causing any user-space pain. 

Fixing `timerfd()` might still be possible. But there is no denying that we would be better off if we could eliminate this kind of API problem before it gets into a stable kernel release and possibly has to be supported for many years. Therein lies the real problem: system calls (and other user-space API features) are being added to the kernel at a high rate, but review of these changes tends to lag behind. Given the difficulty of fixing user-space API mistakes, it would seem that the review standards for API additions should be especially high. Causing that to happen will not be easy, though; reviewer attention is a scarce resource throughout the free software community. 

An idea which has been raised in the past is to explicitly mark new user-space interfaces as being in a volatile "beta" state. For as long as the API remains in that state, the kernel developers are free to change it. Applications would, during this period, rely in the API at their peril. This idea has been rejected in the past, though; it is seen as a way of avoid proper thought ahead of merging a new API into the kernel. Assuming that view still holds, another way will have to be found. 

One part of the solution might well be seen in how the `timerfd()` problems came to light. Michael has demonstrated something your editor has also encountered a number of times: one of the best ways to find shortcomings in an API is to attempt to document it comprehensively. If the kernel community were to resolve that it would not merge user-space API features in the absence of complete documentation, it might just provide the necessary incentive to get that last review pass done. 

This idea seems likely to come up at next month's kernel summit (for which [a preliminary agenda](<http://thunker.thunk.org/pipermail/ksummit-2007-discuss/2007-August/000398.html>) has just been posted). How it will be received is anybody's guess; writing documentation appears to be a task so challenging that even kernel hackers fear to try it. This challenge may be worth taking up, though, if the reward is few long-lasting user-space API problems in the future.
