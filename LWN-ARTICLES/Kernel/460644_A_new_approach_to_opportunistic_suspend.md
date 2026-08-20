---
title: A new approach to opportunistic suspend
url: https://lwn.net/Articles/460644/
date: "September 27, 2011"
category: "Android; Power management-Opportunistic suspend"
author: "By Jonathan Corbet September 27, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
September 27, 2011

"Opportunistic suspend" is the power management technique used by Android devices. Rather than trying to put the various system components into a low-power state, opportunistic suspend works by simply suspending the entire device whenever it is determined that nothing interesting is going on. Managing power consumption in this way is controversial, but the real problem has to do with the determination that it is a good time to suspend. Android's mechanism for controlling opportunistic suspend has been called "wakelocks" or "suspend blockers"; either way, it has had a bumpy ride whenever attempts have been made to merge it into the mainline. John Stultz has just proposed [an alternative to suspend blockers](<https://lwn.net/Articles/460560/>) that is guaranteed an equally bumpy ride, but it is interesting to look at regardless. 

Suspend blockers are a way for either the kernel or user space to tell the system that it is not a good time to suspend; the use of suspend blockers from user space is a privileged operation. To work properly, suspend blockers must be supported by any device that can wake up the system. Drivers for such devices will, when a wakeup event occurs, acquire a suspend blocker and wake any user-space process waiting on the event; once that process reads the event, the suspend blocker will be released. The key is that said user space process, if it is sufficiently privileged, can acquire a suspend blocker of its own before reading the event that woke it up. The overlap between the acquisition of the user-space blocker and the release of the kernel-space blocker allows for reliable system wakeups without the risk of suspending before a wakeup event has been processed. 

One of the things John didn't like about this mechanism was the implicit suspend blocker API between user space and wakeup-capable devices. So he has come up with something a bit different, even if the core idea is quite similar. 

The whole point of suspend blockers is to allow "important" processes to keep the device awake even if it would otherwise choose to suspend itself. So, John asked, why not just mark the important processes as such? His patch adds a new scheduler option: 

```
sched_setscheduler(0, SCHED_STAYAWAKE, &param);
```

Any time that a process has been marked in this way, the kernel simply will not suspend the system. That is true even if the given process blocks; otherwise, a disk I/O operation or page fault would be enough to put the system into an untimely sleep. 

Of course, life is not quite so simple; there are times when it is desirable to suspend the system even with "important" processes on it. One of those is whenever the important process blocks on a device that is capable of generating wakeup events. Enabling suspend at such times requires tweaking the relevant drivers; in essence, a line like: 

```
sched_deboost_task_active_count(current);
```

must be added to remove the blocking process's "important" status just before putting it to sleep. When that device wakes the blocked process, its special status will be returned to it. 

The only remaining problem is, once again, keeping the system from re-suspending before the important process gets its status back. That is handled by placing calls to `__pm_stay_awake()` and `__pm_relax()` around the code that wakes blocked processes. John also had to change the suspend code to make `__pm_stay_awake()` a bit less advisory than it is in current kernels. With that in place, the device will not suspend before any important processes have had the chance to reclaim their status. 

As of this writing, the only [feedback](<https://lwn.net/Articles/460648/>) on the patch set has come from scheduler maintainer Peter Zijlstra. Suffice to say he didn't like it. According to Peter, opportunistic suspend is an attempt to paper over a problem that should be solved in user space; unimportant processes, he says, simply should not be running in the first place. That view, in turn, is unlikely to be popular in the Android camp. So, even though John's approach simplifies the wakelock idea, it does not seem likely to put an end to the debate over that approach to power management.
