---
title: Leaping seconds and looping servers
url: https://lwn.net/Articles/504744/
date: "July 2, 2012"
category: Timers; hrtimer
author: "By Jonathan Corbet July 2, 2012"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
July 2, 2012

As most of the net is likely to have heard by now, Linux servers displayed a notable tendency to misbehave during the leap second event at the end of the day on June 30\. The problem often presented itself as abrupt and sustained load spikes on the affected machines. The bug that caused this behavior has been tracked down (thanks to a determined effort by John Stultz); a look at what happened shines an interesting light on the trickiness of dealing with time in software systems. 

The earth's rotation is slowing over time; contrary to some public claims, this slowing is not caused by Republican administrations, government spending, or proprietary software. In an attempt to keep the official Coordinated Universal Time (UTC) in sync with the earth's behavior, the powers that be occasionally insert an additional second (a "leap second") into a day; 25 such seconds have been inserted since the practice began in 1972\. This habit is not without its detractors, and there are constant calls for its abolition, but, for now, leap seconds are a reality that the world (and the kernel) must deal with. For the curious, [the Wikipedia leap second page](<https://en.wikipedia.org/wiki/Leap_second>) has more detail than almost anybody could want. 

The kernel's core time is kept in a `timespec` structure: 

```
struct timespec {
    	__kernel_time_t	tv_sec;			/* seconds */
    	long		tv_nsec;		/* nanoseconds */
        };
```

It is, in essence, a count of seconds since the beginning of the epoch. Unfortunately, that count is defined to not include leap seconds. So when a leap second happens, the system time must be explicitly corrected; that is done by setting the system clock back one second at the end of that leap second. The code that handles this change is quite old and works pretty much as advertised. It is the source of this message that most Linux systems should have (in some form) in their logs: 

```
Jun 30 19:59:59 dt kernel: Clock: inserting leap second 23:59:60 UTC
```

The kernel's high-resolution timer (hrtimer) code does not use this version of the system time, though — at least, not directly. Instead, hrtimers have a couple of internal time bases that are offset from the system time. These time bases allow the implementation of different clocks; the "realtime" clock should adjust with the time, while the "monotonic" clock must always move forward, for example. Importantly, these timer bases are CPU-specific, since realtime clocks can differ between one CPU and the next in the same system. The hrtimer offsets allow the timer subsystem to quickly turn a system time into a time value appropriate for a specific processor's realtime clock. 

If the system time changes, those offsets must be adjusted accordingly. There is a function called `clock_was_set()` that handles this task. As long as any system time change is followed by a call to `clock_was_set()`, all will be well. The problem, naturally, is that the kernel failed to call `clock_was_set()` after the leap second adjustment, which certainly qualifies as a system time change. So the hrtimer subsystem's idea of the current time moved forward while the system time was held back for a second; hrtimers were thereafter operating one second in the future. The result of that offset is that timers started expiring one second sooner than they should have; that is not quite what the timer developers had in mind when they used the term "high resolution." 

For many applications, having a timer go off one second early is not a big problem. But there are plenty of situations where timers are set for less than one second in the future; all such timers will naturally expire immediately if the timer subsystem is operating one second ahead of the system time. Many of these timers are also recurring timers; they will be re-set immediately after expiration, at which point they will immediately expire again — and so on. The resulting loop is the source of the load spikes reported by victims of this bug across the net. 

The [fix](<https://lwn.net/Articles/504779/>) is to call `clock_was_set()` in the leap second code—a call that had been [removed](<http://git.kernel.org/linus/746976a301ac9c9aa10d7d42454f8d6cdad8ff2b>) in 2007. But it's not quite that simple. The work done by `clock_was_set()` must happen on every CPU, since each CPU has its own set of timer bases. That's not something that can be done in atomic context. So John's patch detects a call in atomic context and defers the work to a workqueue in that case. With this patch in place, the kernel's leap second handling should work again. 

How could such a bug come about? Time-related code is notoriously tricky in general; bugs are common. But the situation is far worse when the code in question is almost never executed. Prior to June 30, 2012, the last leap second was at the end of 2008. That is 3½ years in which the leap second code could have been broken without anybody noticing. If the kernel had a regularly-run regression test that verified the correct functioning of hrtimers in the presence of leap second adjustments, this problem might just have been caught before it affected production systems, but nobody has made a habit of running such tests thus far. 

Perhaps that will change in the future; if nothing else, distributors with support obligations are likely to run some tests ahead of the next scheduled leap second adjustment. Hopefully, that will catch any problems in this particular little piece of code, should they happen to slip in again. Beyond that, one can always hope for an end to leap seconds. The kernel could also contemplate a switch to [international atomic time (TAI)](<https://en.wikipedia.org/wiki/International_Atomic_Time>), which does not have leap seconds, for its internal representation. Using TAI internally has its own challenges, though, including a need to avoid changing the time representation as seen by user space—meaning that the kernel would still have to track leap seconds internally. So it seems likely that, one way or another, leap seconds are likely to continue to be a source of irritation and bugs in the future.
