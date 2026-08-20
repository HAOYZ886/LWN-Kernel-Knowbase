---
title: Clockevents and dyntick
url: https://lwn.net/Articles/223185/
date: "February 21, 2007"
category: Clockevents; Dynamic tick; hrtimer
author: 
---

One of the last patch sets to be merged before the 2.6.21 window closed was the clockevents and dyntick work from the real-time tree. These patches have been in the works for some time, and were originally targeted for merging in 2.6.19. In the process, the developers (primarily Ingo Molnar and Thomas Gleixner) discovered one of the fundamental laws of kernel development: if your patches break Andrew Morton's laptop, they are unlikely to make it into the mainline. That little difficulty has now been overcome, with the result that 2.6.21 will include some interesting core changes. 

Dealing with clock devices has traditionally been handled in the kernel's architecture-specific code. The result has been a lot of duplicated code between architectures (there are more architectures than common timer devices) and no uniform interface for the core kernel to make use of these devices. John Stultz's generic time of day infrastructure resolved a number of those problems, at least for the timekeeping task, but anybody who wanted to program timer devices in a more general way still ended up dealing with architecture-specific code. 

The "clockevents" patch set finishes this job. At its core, clockevents creates a driver API for devices which can deliver interrupts at a specific time in the future. The API tracks the capabilities of each timer (resolution and whether it can do one-shot or periodic interrupts, for example) and provides a simple interface for arming the timer. This API is defined in the core kernel, with only a low-level driver remaining in the architecture-specific code. The end result is that the kernel now has the means to query and use timer capabilities in an architecture-independent manner. 

With the clockevents mechanism in place, it becomes possible to support truly high-resolution timers. When such a timer is requested, all that is required is to pick a suitable clockevent device and arm it for the desired time. These devices can deliver interrupts with a high degree of precision, with the result that kernel timers, too, can offer high precision - a feature which is of clear utility to real-time users (among others). 

The periodic timer tick is now implemented with a clockevent as well. It does all of the things the old timer-based interrupt did - updating `jiffies`, accounting CPU time, etc. - but it is run out of the new infrastructure. 

All of this is an improvement, but there is still one thing which could be better: there is no real need for a periodic tick in the system. That is especially true when the processor is idle. An idle CPU can save quite a bit of power, but waking that CPU up 100 times (or more) per second will hurt those power savings considerably. With a flexible timer infrastructure, there is no point in turning the CPU back on until it has something to do. So, when the (i386) kernel goes into its idle loop, it checks the next pending timer event. If that event is further away than the next tick, the periodic tick is turned off altogether; instead, the timer is is programmed to fire when the next event comes due. The CPU can then rest unharrassed until that time - unless an interrupt comes in first. Once the processor goes out of the idle state, the periodic tick is restored. 

What's in 2.6.21 is, thus, not a full dynamic tick implementation. Eliminating the tick during idle times is a good step forward, but there is value in getting rid of the tick while the system is running as well - especially on virtualized systems which may be sharing a host with quite a few other clients. The dynamic tick documentation file suggests that the developers have this goal in mind: 

The implementation leaves room for further development like full tickless systems, where the time slice is controlled by the scheduler, variable frequency profiling, and a complete removal of jiffies in the future. 

So expect some interesting work in the future - the removal of `jiffies` alone has a number of interesting implications. The developers also have support for the x86_64 and ARM architectures, though that support has not been merged for 2.6.21; MIPS and PowerPC support is in the works as well.
