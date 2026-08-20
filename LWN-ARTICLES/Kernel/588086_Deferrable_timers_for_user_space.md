---
title: Deferrable timers for user space
url: https://lwn.net/Articles/588086/
date: "February 26, 2014"
category: Timers
author: "By Jonathan Corbet February 26, 2014"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
February 26, 2014

The [deferrable timers](<https://lwn.net/Articles/228143/>) concept in the kernel dates back to 2007. A deferrable timer can be used in situations where an arbitrary delay between the timer's expiration and the execution of the timeout code is acceptable. In such situations, the expiration of the timer can be delayed until something else wakes up the CPU. Deferring expiration in this way minimizes the number of wakeups, which, in turn, helps to minimize power consumption. 

Deferrable timers are available within the kernel, but they are not provided to user space. So the timer-related system calls (including `timerd_settime()`, `clock_nanosleep()`, and `nanosleep()`) will all make an effort to notify user space quickly on timer expiration, even if user space could tolerate some delay. That is irritating to developers who are working to improve power performance. The good news for those developers is that, after a couple of false starts, it now appears that deferrable timers may finally be made available to user space. 

Some readers will certainly be thinking about the [timer slack](<https://lwn.net/Articles/369549/>) mechanism, which _is_ available in user space. However, it affects all timers; for some applications, some timers may be more deferrable than others. Timers with slack can also only be deferred for so long, meaning that they might still wake a sleeping CPU when they expire. So deferrable timers may well address some use cases that are not well handled by timer slack. 

Back in 2012, Anton Vorontsov sent out [a patch set](<https://lwn.net/Articles/514707/>) adding deferrable timer support to the `timerfd_settime()` system call. In putting together this patch, Anton ran into a problem: like all of the timing-related system calls, the timerfd mechanism uses the kernel's high-resolution timers. But high-resolution timers do not support deferrable operation; that functionality is limited to the older "timer wheel" mechanism (described in [this article](<https://lwn.net/Articles/152436/>)). The timer wheel is old code with a number of problems; it has been slated for removal for years. But nobody has done that work, so the timer wheel remains in place, and it remains the only facility with the deferrable option. 

Anton's response was to split timers in the timerfd subsystem across both mechanisms. Regular timer requests would use the high-resolution timer subsystem as usual, but any request with the `TFD_TIMER_DEFERRABLE` flag set would, instead, be handled by the timer wheel. Among other things, that limited the resolution of deferrable timers to one jiffy (0.001 to 0.01 seconds, depending on the kernel configuration), but, if the timer is deferrable, low resolution is not necessarily a problem. Still, this patch set did not go very far, and Anton appears to have dropped it fairly quickly. 

More recently, Alexey Perevalov has picked up this concept and tried to push it forward. He first [posted a patch set](<https://lwn.net/Articles/588127/>) in January; this posting generated rather more discussion than its predecessor did. John Stultz was [concerned](<https://lwn.net/Articles/588128/>) that only timerfd timers gained the new functionality; it would be better, he thought, to push it down to a lower level so that all timer-related system calls would benefit. Doing so, he thought, would likely entail adding the deferrable capability to the high-resolution timer subsystem. 

Thomas Gleixner was rather more emphatic, [stating](<https://lwn.net/Articles/588131/>) that use of the timer wheel ""is not going to happen"". He suggested that this functionality should instead be added to high-resolution timers in the form of a new set of clock IDs. The clock ID is a parameter provided by user space describing which timekeeping regime should be used. `CLOCK_REALTIME`, for example, corresponds to the system clock; it can go backward should the administrator change the system time. The `CLOCK_MONOTONIC` clock, instead, is guaranteed to always move forward. There are a number of other clocks, including `CLOCK_TAI`, added in 3.10, which corresponds to international atomic time. Thomas tossed out a proof-of-concept patch adding new versions of all of these clocks (`CLOCK_MONOTONIC_DEFERRABLE`, for example) that would provide deferrable operation. 

John, however, [argued](<https://lwn.net/Articles/588133/>) that clock IDs were not the right interface to expose to user space: 

My reasoning was that the deferrablity isn't a clock domain, and is more of a modifier. Thus to keep the interfaces somewhat sane (and avoiding having to add N new clockids for each new modifier), we should utilize the flag arguments to timers. So instead of just having TIMER_ABSTIME, we could add TIMER_DEFER, etc, which we could utilize instead. 

The use of separate clock IDs within the kernel as an implementation detail is fine, he said, but the right way for user space to request the feature is with modifier flags. Fortunately, almost all of the relevant system calls already have flag arguments; the big exception is `nanosleep()`, which is a call some developers would like to see simply vanish anyway. John's argument, when expressed in this way, prevailed with no real dissent. 

Alexey posted a couple more versions of his patch set, but, to put it gently, they did not find approval with Thomas, who eventually posted [a deferrable timers patch set](<https://lwn.net/Articles/587581/>) of his own, showing how he thinks the problem should be solved. In this patch set, `clock_nanosleep()` gets a new `TIMER_IS_DEFERRABLE` flag, while `timerfd_settime()` gets `TFD_TIMER_IS_DEFERRABLE`. In either case, setting that flag causes the kernel to use one of the new deferrable internal clock IDs. Timers on those IDs never actually program the hardware, so they never generate interrupts and cannot wake the system. Instead, the expiration functions will be executed when the system is awakened for some other task. In the absence of the new flag, these system calls behave as they always have. 

Thomas's patch set has not gotten much in the way of comments beyond low-level implementation details; if that situation persists for long, silence is likely to indicate consent. So, unless some surprise comes along, the kernel will probably offer deferrable timers to user space around 3.15 or so.
