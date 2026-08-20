---
title: Time namespaces
url: https://lwn.net/Articles/766089/
date: "September 21, 2018"
category: Namespaces
author: "By Jonathan Corbet September 21, 2018"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 21, 2018

The kernel's [namespace](<https://lwn.net/Articles/766124/>) abstraction allows different groups of processes to have different views of the system. This feature is most often used with containers; it allows each container to have its own view of the set of running processes, the network environment, the filesystem hierarchy, and more. One aspect of the system that remains universal, though, is the concept of the system time. The recently posted [time namespace patch set](<https://lwn.net/ml/linux-kernel/20180919205037.9574-1-dima@arista.com/>) (from Dmitry Safonov with a lot of work by Andrei Vagin) seeks to change that. 

Creating a virtualized view of the system time is not a new concept; Jeff Dike [posted an implementation](<https://lwn.net/Articles/180375/>) back in 2006 to support his user-mode Linux project. Those patches were not merged at the time but, since then, the use of containers has taken off and the interest has increased. One might view time as a universal concept, but there are use cases for a per-container notion of time; they can be as simple as testing software at different points in time. The driving force behind this patch set, though, is likely to be problems associated with the checkpointing of processes and migrating them between physical hosts. When a process is restarted, it should have a consistent view of time, and that may require applying some adjustments at restart time. 

The implementation is straightforward enough. Each time namespace contains a set of offsets to be added to the system's notion of the current time. The kernel maintains a number of clocks with different characteristics (documented [here](<http://man7.org/linux/man-pages/man2/timerfd_create.2.html>)), each of which can have a different offset. Some of these clocks, such as `CLOCK_MONOTONIC`, have an undefined start point that will vary from one running system to the next, so they will need their own offsets to maintain consistent behavior for a container that has been migrated. System calls that adjust the system time will, when called outside of the root time namespace, adjust the namespace-specific offsets instead. 

There is one small complication, in that some of the time-related system calls are implemented as [virtual system calls](<https://lwn.net/Articles/615809/>) on some architectures for performance reasons. Querying the current time can be a frequent operation, so it can be worth the trouble to answer such queries without actually entering the kernel. Making the virtual system calls aware of time namespaces requires making the clock offsets available to user space; the good news is that there is a small piece of the address space called the "VVAR page" (even though it is larger than one page) meant to hold just this kind of data. The time namespace work adds another page to this VVAR region to hold the time offsets, allowing calls like `gettimeofday()` to continue to work without entering the kernel. 

Namespace maintainer Eric Biederman has [expressed support](<https://lwn.net/ml/linux-kernel/874lej6nny.fsf@xmission.com/>) for time namespaces, but he has also suggested some changes. His observation is that the [`timekeeper` structure](<https://elixir.bootlin.com/linux/latest/source/include/linux/timekeeper_internal.h#L45>) used within the kernel to implement the various clocks already contains a set of offsets relating those clocks to the hardware's idea of the current time. Rather than adding a second layer of offsets, he suggested, each namespace could be given its own `timekeeper` structure and the offsets found there could be tweaked instead. That might add to the complexity of the implementation, but this approach would have some advantages. Most of the kernel's current timekeeping code would just work with namespaces, allowing better testing overall with fewer special cases. Integrating namespaces at this level would also allow each container to run its own NTP process, and different containers could, for example, use different leap-second policies. 

Biederman raised the possibility of security issues if time namespaces can be used to manipulate dates on files in filesystems, though he was not sure if that actually mattered. He also suggested that access to the realtime clock (the hardware clock that, in the end, drives the system's timekeeping) should perhaps be left out of the time namespace until it is clear that there are actual use cases for it. If that use case does arise, he said, some thought will have to be given to how the realtime clock, which is a global resource, should be presented to non-root namespaces. 

There are, in other words, a few details remaining to be worked out regarding how time namespaces will work. There do not, however, appear to be any real obstacles to a solution, so chances are good that the kernel's collection of namespaces will be enhanced by time namespaces sometime in the not-too-distant future. Given how long the idea has been around, one might say it's about time.
