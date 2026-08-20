---
title: Timer slack
url: https://lwn.net/Articles/369549/
date: "January 12, 2010"
category: Timers
author: "By Jonathan Corbet January 12, 2010"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
January 12, 2010

One of the best ways to reduce a system's power usage is to avoid waking up the CPU whenever possible. Minimizing wakeups, in turn, is facilitated by ensuring that timers expire at the same time when it makes sense to do so. Waking the processor once to handle two timers is much more efficient than handling them in two separate wakeups. But doing so typically requires adjusting expiration times. For standard (not high resolution) kernel timers, the only way to make that adjustment is with the `round_jiffies()` function, which makes timeout periods coarser in the hopes that they will coincide more often. This method works to an extent, but it requires code changes wherever timers are used. 

Arjan van de Ven has proposed an enhancement to the timer API - called [timer slack](<http://lwn.net/Articles/369361/>) \- which should make it easier to coalesce timer events. In essence, it adds a certain amount of fuzziness to timer expiration times, giving the kernel some flexibility in how the timers are scheduled. That fuzziness is set with: 

```
void set_timer_slack(struct timer_list *timer, int slack_hz);
```

In essence, this call says that any timeout scheduled with the given `timer` can be delayed by up to `slack_hz` jiffies. By default, the slack is set to 0.4% of the total timeout period - a very conservative value. When the timer is queued, the actual expiration time is determined by means of a simple algorithm to choose a well-defined time within the slack period. 

The value of this approach is that it makes it easy to coalesce timer events from multiple sources without needing to change every call site. Additional flexibility can then be had by increasing the slack for specific, frequently-used timers, but, even without that, slack timers should improve power efficiency on many systems.
