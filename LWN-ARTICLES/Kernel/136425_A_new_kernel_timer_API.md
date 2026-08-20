---
title: A new kernel timer API
url: https://lwn.net/Articles/136425/
date: "May 18, 2005"
category: Timers
author: 
---

John Stultz's new core time subsystem was covered on this page [back in January](<https://lwn.net/Articles/120850/>). This patch set, which will be submitted soon for inclusion (into -mm), replaces a mess of architecture-specific time implementations with a cleaner, central time subsystem which can take full advantage of hardware time sources. Nishanth Aravamudan would now like to take advantage of the new low-level time code by replacing the kernel timer implementation. This work, if accepted, will lead to the incorporation of a new timer API to be used by kernel code when a function must be called at some point in the future. 

In current Linux kernels, internal time (for most purposes) is measured in "`jiffies`," which are really just a counter which is incremented when each timer interrupt happens. The new time code supersedes `jiffies` with an absolute, monotonically increasing count of nanoseconds. References to `jiffies` thus become a call to: 

```
nsec_t do_monotonic_clock(void);
```

Using nanoseconds allows kernel code to work with high-resolution time in real-world units. That, in turn, lets kernel developers forget about the (error-prone) conversions between `jiffies` and real-world time which are currently necessary. 

Nishanth's add-on patch changes the timer subsystem to use nanoseconds as well. The current `add_timer()` and `mod_timer()` interfaces remain supported, but are deprecated. The new interface for setting (or modifying) a timer is: 

```
int set_timer_nsecs(struct timer_list *timer, nsec_t expires);
        void set_timer_on_nsecs(struct timer_list *timer, nsec_t expires, 
                                int cpu);
```

This function will cause the given `timer` to be set to go off at `expires`, which is an absolute nanoseconds count. Usually, `expires` will be calculated by adding the desired delay (in nanoseconds) to whatever `do_monotonic_clock()` returns. 

It's worth noting that this patch changes the meaning of the `expires` field in the `timer_list` structure. This field is now represented in an internal "timer intervals" unit, rather than in `jiffies`. If the old `add_timer()` and `mod_timer()` interfaces are used, the `expires` field will be silently converted to the internal format. Code which performs calculations on `expires` (by increasing the delay and calling `mod_timer()`, for example) could be in for a surprise. 

This patch also deprecates `schedule_timeout()`, in favor of these functions: 

```
nsec_t schedule_timeout_nsecs(nsec_t timeout);
        unsigned long schedule_timeout_usecs(unsigned long usecs);
        unsigned int schedule_timeout_msecs(unsigned int msecs);
```

All three of these functions will set a timer for the given delay (which is a relative value, not absolute), then call `schedule()`.
