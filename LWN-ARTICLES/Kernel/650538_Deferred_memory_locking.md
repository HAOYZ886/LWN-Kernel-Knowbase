---
title: Deferred memory locking
url: https://lwn.net/Articles/650538/
date: "July 8, 2015"
category: "Memory management-Page locking"
author: "By Jonathan Corbet July 8, 2015"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
July 8, 2015

The [`mlock()` and `mlockall()` system calls](<http://man7.org/linux/man-pages/man2/mlock.2.html>) are charged with locking a portion (or all) of a process's address space into physical memory. The most common use cases for this functionality are situations where the latency of a page fault cannot be afforded and protecting sensitive data (cryptographic keys, say) from being written out to the swap device. Both system calls assume that the caller wants all of the memory present and locked immediately, but that may not always be the case. As a result, we are likely to see new versions of the memory-locking system calls in the near future. 

The idea that a user who has requested the locking of a range of memory doesn't actually want it locked now may seem a little strange; that is what `mlock()` and `mlockall()` were created for, after all. The problem with immediate locking, as described by Eric Munson in [his patch set](<https://lwn.net/Articles/650410/>), is that faulting in and locking a large address range can take a long time, and much of that time may be wasted if the calling process never actually uses much of that memory. If the cost of a page fault on the first access to a given page is not an issue, deferring the population and locking of a memory range can be a useful way to improve performance. 

The cryptographic use case is one where deferred locking might make sense: the buffer to be locked may need to be able to handle a large worst case, but, most of the time, the portion of the buffer that's actually used is quite a bit smaller. If the pages that make up that buffer could only be locked after they are first faulted in, the objective of preventing writeout to the swap device will be met with lower overhead overall. Eric also mentions programs that use small parts of a large buffer, but which cannot know from the outset which parts will be used. 

The solution in both cases is to modify `mlock()` so that it does not fault in all of the pages in the indicated address range. Instead, the range is simply marked as "lock on fault." Whenever a page within that range is faulted in, it will be locked from then on. 

The problem is that `mlock()` has this prototype: 

```
int mlock(const void *addr, size_t len);
```

There is no way to tell the kernel to not fault the pages in immediately. The natural response is to create a new system call that has a feature that [arguably should have been present](<https://lwn.net/Articles/585415/>) in `mlock()` in the first place: a "flags" argument: 

```
int mlock2(const void *addr, size_t len, int flags);
```

The `flags` argument has two possibilities: `MLOCK_LOCKED` (to fault in the pages immediately) or `MLOCK_ONFAULT` (which only locks pages once they are faulted in). Exactly one of those flags must be present in any `mlock2()` call. 

The `mlockall()` system call _does_ already have a flags argument; the new `MCL_ONFAULT` flag has been added to request the new behavior via that interface. There is also a new flag (`MAP_LOCKONFAULT`) that can be used to get locked-on-fault behavior when creating an address range with `mmap()`. 

Eric's patch set adds new versions of the corresponding unlock system calls: 

```
int munlock2(const void *addr, size_t len, int flags);
        int munlockall2(int flags);
```

These system calls have the effect of clearing the given `flags`; the actual unlocking of memory is a side effect if all the flags are cleared. If a region has been locked with `MLOCK_ONFAULT`, one can call: 

```
munlock2(addr, len, MLOCK_ONFAULT);
```

to cancel the on-fault locking in the future while leaving currently locked pages in place, or: 

```
munlock2(addr, len, MLOCK_LOCKED|MLOCK_ONFAULT);
```

to unlock the address range entirely. It is not entirely clear (to your editor, at least) what will happen if `munlock2()` is called with just the `MLOCK_LOCKED` flag in this situation. Similar things can be done with `munlockall2()`; in this case, it is also possible to clear existing flags like `MCL_FUTURE`. 

This patch set has been through a few iterations over the last few months. It has taken Eric a bit of work to convince reviewers of the value of this functionality; review comments also led to the addition of the new system calls (as opposed to just the new `mmap()` and `mlockall()` flags). This patch set has found its way into the -mm tree, which is a good sign that it's likely to head toward the mainline sometime in the relatively near future.
