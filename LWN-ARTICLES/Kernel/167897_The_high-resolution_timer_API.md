---
title: "The high-resolution timer API"
url: https://lwn.net/Articles/167897/
date: "January 16, 2006"
category: Timers; hrtimer
author: 
---

Last September, this page featured [an article on the ktimers patch](<http://lwn.net/Articles/152436/>) by Thomas Gleixner. The new timer abstraction was designed to enable the provision of high-resolution timers in the kernel and to address some of the inefficiencies encountered when the current timer code is used in this mode. Since then, there has been a large amount of discussion, and the code has seen significant work. The end product of that work, now called "hrtimers," was merged for the 2.6.16 release. 

At its core, the hrtimer mechanism remains the same. Rather than using the "timer wheel" data structure, hrtimers live on a time-sorted linked list, with the next timer to expire being at the head of the list. A separate red/black tree is also used to enable the insertion and removal of timer events without scanning through the list. But while the core remains the same, just about everything else has changed, at least superficially. 

There is a new type, `ktime_t`, which is used to store a time value in nanoseconds. This type, found in `<linux/ktime.h>`, is meant to be used as an opaque structure. And, interestingly, its definition changes depending on the underlying architecture. On 64-bit systems, a `ktime_t` is really just a 64-bit integer value in nanoseconds. On 32-bit machines, however, it is a two-field structure: one 32-bit value holds the number of seconds, and the other holds nanoseconds. The order of the two fields depends on whether the host architecture is big-endian or not; they are always arranged so that the two values can, when needed, be treated as a single, 64-bit value. Doing things this way complicates the header files, but it provides for efficient time value manipulation on all architectures. 

A whole set of functions and macros has been provided for working with `ktime_t` values, starting with the traditional two ways to declare and initialize them: 

```
DEFINE_KTIME(name);   /* Initialize to zero */
    
        ktime_t kt;
        kt = ktime_set(long secs, long nanosecs);
```

Various other functions exist for changing `ktime_t` values; all of these treat their arguments as read-only and return a `ktime_t` value as their result: 

```
ktime_t ktime_add(ktime_t kt1, ktime_t kt2);
        ktime_t ktime_sub(ktime_t kt1, ktime_t kt2);  /* kt1 - kt2 */
        ktime_t ktime_add_ns(ktime_t kt, u64 nanoseconds);
```

Finally, there are some type conversion functions: 

```
ktime_t timespec_to_ktime(struct timespec tspec);
        ktime_t timeval_to_ktime(struct timeval tval);
        struct timespec ktime_to_timespec(ktime_t kt);
        struct timeval ktime_to_timeval(ktime_t kt);
        clock_t ktime_to_clock_t(ktime_t kt);
        u64 ktime_to_ns(ktime_t kt);
```

The interface for hrtimers can be found in `<linux/hrtimer.h>`. A timer is represented by `struct hrtimer`, which must be initialized with: 

```
void hrtimer_init(struct hrtimer *timer, clockid_t which_clock);
```

Every hrtimer is bound to a specific clock. The system currently supports two clocks, being: 

  * `CLOCK_MONOTONIC`: a clock which is guaranteed always to move forward in time, but which does not reflect "wall clock time" in any specific way. In the current implementation, `CLOCK_MONOTONIC` resembles the `jiffies` tick count in that it starts at zero when the system boots and increases monotonically from there. 

  * `CLOCK_REALTIME` which matches the current real-world time. 

The difference between the two clocks can be seen when the system time is adjusted, perhaps as a result of administrator action, tweaking by the network time protocol code, or suspending and resuming the system. In any of these situations, `CLOCK_MONOTONIC` will tick forward as if nothing had happened, while `CLOCK_REALTIME` may see discontinuous changes. Which clock should be used will depend mainly on whether the timer needs to be tied to time as the rest of the world sees it or not. The call to `hrtimer_init()` will tie an hrtimer to a specific clock, but that clock can be changed with: 

```
void hrtimer_rebase(struct hrtimer *timer, clockid_t new_clock);
```

Most of the `hrtimer` fields should not be touched. Two of them, however, must be set by the user: 

```
int  (*function)(void *);
        void *data;
```

As one might expect, `function()` will be called when the timer expires, with `data` as its parameter. 

Actually setting a timer is accomplished with: 

```
int hrtimer_start(struct hrtimer *timer, ktime_t time,
                          enum hrtimer_mode mode);
```

The `mode` parameter describes how the `time` parameter should be interpreted. A `mode` of `HRTIMER_ABS` indicates that `time` is an absolute value, while `HRTIMER_REL` indicates that `time` should be interpreted relative to the current time. 

Under normal operation, `function()` will be called after (at least) the requested expiration time. The hrtimer code implements a shortcut for situations where the sole purpose of a timer is to wake up a process on expiration: if `function()` is `NULL`, the process whose task structure is pointed to by `data` will be awakened. In most cases, however, code which uses hrtimers will provide a callback `function()`. That function has an integer return value, which should be either `HRTIMER_NORESTART` (for a one-shot timer which should not be started again) or `HRTIMER_RESTART` for a recurring timer. 

In the restart case, the callback must set a new expiration time before returning. Usually, restarting timers are used by kernel subsystems which need a callback at a regular interval. The hrtimer code provides a function for advancing the expiration time to the next such interval: 

```
unsigned long hrtimer_forward(struct hrtimer *timer, ktime_t interval);
```

This function will advance the timer's expiration time by the given `interval`. If necessary, the interval will be added more than once to yield an expiration time in the future. Generally, the need to add the interval more than once means that the system has overrun its timer period, perhaps as a result of high system load. The return value from `hrtimer_forward()` is the number of missed intervals, allowing code which cares to detect and respond to the situation. 

Outstanding timers can be canceled with either of: 

```
int hrtimer_cancel(struct hrtimer *timer);
        int hrtimer_try_to_cancel(struct hrtimer *timer);
```

When `hrtimer_cancel()` returns, the caller can be sure that the timer is no longer active, and that its expiration function is not running anywhere in the system. The return value will be zero if the timer was not active (meaning it had already expired, normally), or one if the timer was successfully canceled. `hrtimer_try_to_cancel()` does the same, but will not wait if the timer function is running; it will, instead, return -1 in that situation. 

A canceled timer can be restarted by passing it to `hrtimer_restart()`. 

Finally, there is a small set of query functions. `hrtimer_get_remaining()` returns the amount of time left before a timer expires. A call to `hrtimer_active()` returns nonzero if the timer is currently on the queue. And a call to: 

```
int hrtimer_get_res(clockid_t which_clock, struct timespec *tp);
```

will return the true resolution of the given clock, in nanoseconds.
