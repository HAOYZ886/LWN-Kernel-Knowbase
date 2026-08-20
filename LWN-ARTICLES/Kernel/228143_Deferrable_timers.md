---
title: Deferrable timers
url: https://lwn.net/Articles/228143/
date: "March 28, 2007"
category: Dynamic tick; Timers
author: 
---

The [dynamic tick](<http://lwn.net/Articles/223185/>) code featured in the upcoming 2.6.21 kernel seeks to avoid processor wakeups by turning off the period timer tick when nothing is happening. Before stopping the clock, the kernel must decide when it should wake up again; this decision involves looking at the timer queue to see when the next timer expires. In the absence of other events (hardware interrupts, for example), the system will sleep until the nearest timer is due. 

Many of these timers should, in fact, run as soon as the requested period has expired. Others, however, are less important - to the point that they are not worth waking up the processor. These non-critical timeouts can run some fraction of a second later (when the processor wakes up for other reasons) and nobody will notice the difference. So it would be nice if there were a way to tell the kernel that a specific timer does not require immediate action on expiration and that the processor should not wake up for the sole purpose of handling it. 

Venki Pallipadi has created such a way with the [deferrable timers patch](<https://lwn.net/Articles/228147/>). There is just one new function added to the internal kernel API: 

```
void init_timer_deferrable(struct timer_list *timer);
```

Timers which are initialized in this fashion will be recognized as deferrable by the kernel. They will not be considered when the kernel makes its "when should the next timer interrupt be?" decision. When the system is busy these timers will fire at the scheduled time. When things are idle, instead, they will simply wait until something more important wakes up the processor. 

Venki appears to have gone to great length to minimize the changes required by this patch. So, in particular, the `timer_list` structure does not change at all. Instead, the low-order bit on an internal pointer (which is known to always be zero) is repurposed as a "deferrable" flag. The result is that the `timer_list` structure does not grow to support this new functionality, at the cost of requiring all code using the internal `base` pointer to mask out the "deferrable" bit. 

The patch, as presented, only affects timers used within the kernel; no code has been changed to actually use deferrable timers yet. There could be potential in extending this interface somehow to user space. Our user space remains full of applications which feel the need to wake up frequently to check the state of the world; these applications are a real problem for power-limited systems. If those applications truly cannot be fixed, perhaps they could at least indicate a willingness to wait when nothing important is going on.
