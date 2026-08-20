---
title: "High- (but not too high-) resolution timeouts"
url: https://lwn.net/Articles/296578/
date: "September 2, 2008"
category: Scheduler; Timers; hrtimer
author: "By Jonathan Corbet September 2, 2008"
---

By **Jonathan Corbet**  
September 2, 2008

Linux provides a number of system calls that allow an application to wait for file descriptors to become ready for I/O; they include `select()`, `pselect()`, `poll()`, `ppoll()`, and `epoll_wait()`. Each of these interfaces allows the specification of a timeout putting an upper bound on how long the application will be blocked. In typical fashion, the form of that timeout varies greatly. `poll()` and `epoll_wait()` take an integer number of milliseconds; `select()` takes a `struct timeval` with microsecond resolution, and `ppoll()` and `pselect()` take a `struct timespec` with nanosecond resolution. 

They are all the same, though, in that they convert this timeout value to jiffies, with a maximum resolution between one and ten milliseconds. A programmer might program a `pselect()` call with a 10 nanosecond timeout, but the call may not return until 10 milliseconds later, even in the absence of contention for the CPU. An error of six orders of magnitude seems like a bit much, especially given that contemporary hardware can easily support much more accurate timing. 

Arjan van de Ven recently surfaced with [a patch set](<http://lwn.net/Articles/296398/>) aimed at addressing this problem. The core idea is simple: have the code implementing `poll()` and `select()` use high-resolution timers instead of converting the timeout period to low-resolution jiffies. The implementation relied on a new function to provide the timeouts: 

```
long schedule_hrtimeout(struct timespec *time, int mode);
```

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Here, `time` is the timeout period, as interpreted by `mode` (which is either `HRTIMER_MODE_ABS` or `HRTIMER_MODE_REL`). 

High-resolution timeouts are a nice feature, but one can immediately imagine a problem: higher-resolution timeouts are less likely to coincide with other events which wake up the processor. The result will be more wakeups and greater power consumption. As it happens, there are few developers who are more aware of this fact than Arjan, who has done quite a bit of work aimed at keeping processors asleep as much as possible. His solution to this problem was to only use high-resolution timeouts if the timeout period is less than one second. For longer timeout periods, the old, jiffie-based mechanism was used as before. 

Linus [didn't like that solution](<https://lwn.net/Articles/296581/>), calling it "ugly." His preference, instead, was to have `schedule_hrtimeout()` apply an appropriate amount of fuzz to all timeout values; the longer the timeout, the less resolution would be supplied. Alan Cox [suggested](<https://lwn.net/Articles/296585/>) that a better mechanism would be for the caller to supply the required accuracy with the timeout value. The problem with that idea, as Linus pointed out, is that the current system call interfaces provide no way for an application to supply the accuracy value. One could create more `poll()`-like system calls - as if there weren't enough of them already - with an accuracy parameter, but that looks like a lot of trouble to create a non-standard interface which few programmers would bother to use. 

A different solution came in the form of Arjan's [range-capable timer patch set](<http://lwn.net/Articles/296548/>). This patch extends hrtimers to accept two timeout values, called the "soft" and "hard" timeouts. The soft value - the shorter of the two - is the first time at which the timeout can expire; the kernel will make its best effort to ensure that it does not expire after the hard period has elapsed. In between the two, the kernel is free to expire the timer at any convenient time. 

It's a useful feature, but it comes at the cost of some significant API changes. To begin with, the `expires` field of `struct hrtimer` goes away. Rather than manipulate `expires` directly, kernel code must now use one of the new accessor functions: 

```
void hrtimer_set_expires(struct hrtimer *timer, ktime_t time);
        void hrtimer_set_expires_tv64(struct hrtimer *timer, s64 tv64);
        void hrtimer_add_expires(struct hrtimer *timer, ktime_t time);
        void hrtimer_add_expires_ns(struct hrtimer *timer, unsigned long ns);
        ktime_t hrtimer_get_expires(const struct hrtimer *timer);
        s64 hrtimer_get_expires_tv64(const struct hrtimer *timer);
        s64 hrtimer_get_expires_ns(const struct hrtimer *timer);
        ktime_t hrtimer_expires_remaining(const struct hrtimer *timer);
```

Once that's done, the range capability is added to hrtimers. By default, the soft and hard expiration times are the same; code which wishes to set them independently can use the new functions: 

```
void hrtimer_set_expires_range(struct hrtimer *timer, ktime_t time, 
                                       ktime_t delta);
        void hrtimer_set_expires_range_ns(struct hrtimer *timer, ktime_t time,
                                          unsigned long delta);
        ktime_t hrtimer_get_softexpires(const struct hrtimer *timer);
        s64 hrtimer_get_softexpires_tv64(const struct hrtimer *timer)
```

In the new "set" functions, the specified `time` is the soft timeout, while `time+delta` provides the hard timeout value. There is also another form of `schedule_timeout()`: 

```
int schedule_hrtimeout_range(ktime_t *expires, unsigned long delta,
    				 const enum hrtimer_mode mode);
```

With this infrastructure in place, `poll()` and friends can be given approximate timeouts; the only remaining question is just how wide the range of times should be. In Arjan's patch, that range comes from two different sources. The first is a new field in the task structure called `timer_slack_ns`; as one might expect, it specifies the maximum expected timer accuracy in nanoseconds. This value can be adjusted via the `prctl()` system call. The default value is set to 50 microseconds - approximate to a certain degree, but still far more accurate than the timeouts in current kernels. 

Beyond that, though, there is a heuristic function which provides an accuracy value depending on the requested timeout period. In the case of especially long timeouts - more than ten seconds - the accuracy is set to 100ms; as the timeouts get shorter, the amount of acceptable error drops, down to a minimum of 10ns for very brief timeouts. Normally, `poll()` and company will use the value returned by the heuristic, but with the exception that the accuracy will never exceed the value found in `timer_slack_ns`. 

The end result is the provision of more accurate timeouts on the polling functions while, simultaneously, preserving the ability to combine timeouts with other system events.
