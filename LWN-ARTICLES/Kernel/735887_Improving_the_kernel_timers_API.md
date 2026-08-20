---
title: Improving the kernel timers API
url: https://lwn.net/Articles/735887/
date: "October 9, 2017"
category: "Security-Kernel hardening; Timers"
author: "By Jonathan Corbet October 9, 2017"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
October 9, 2017

The kernel's timer interface has been around for a long time, and its API shows it. Beyond a lack of conformance with current in-kernel interface patterns, the timer API is not as efficient as it could be and stands in the way of ongoing kernel-hardening efforts. A late addition to the 4.14 kernel paves the way toward a wholesale change of this API to address these problems. 

It is worth noting that the kernel has two core timer mechanisms. One of those — the high-resolution timer (or "hrtimer") — subsystem, is focused on near-term events where the timer is expected to run to completion. The other subsystem is just called "kernel timers"; it offers less precision but is more efficient in situations where the timer will probably be canceled before it fires. There are many places in the kernel where timers are used to detect when a device or a network peer has failed to respond within the expected time; when, as usual, the expected response _does_ happen, the timer is canceled. Kernel timers are well suited to that kind of use. The work at hand focuses on that second type of timer. 

Kernel timers are described by the `timer_list` structure, defined in [`<linux/timer.h>`](<http://elixir.free-electrons.com/linux/v4.14-rc4/source/include/linux/timer.h>): 

```
struct timer_list {
    	unsigned long		expires;
    	void			(*function)(unsigned long);
    	unsigned long		data;
    	/* ... other stuff elided ... */
        }
```

The `expires` field contains the expiration time of the timer (in jiffies); on expiration, `function()` will be called with the given `data` value. It is possible to fill in a `timer_list` structure manually, but it is more common to use the `setup_timer()` macro: 

```
void setup_timer(timer, function, data);
```

There are a number of issues with this API, as [argued by Kees Cook](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=686fef928bba6be13cabe639f154af7d72b63120>). The `data` field bloats the `timer_list` structure unnecessarily and, as an unadorned `unsigned long` value, it resists any sort of type checking. It is not uncommon for callers to cast pointer values to and from this value, for example. For these reasons, it is far more common in current kernel APIs to dispense with the `data` field and, instead, just pass a pointer to the relevant structure (the `timer_list` structure in this case) to the callback. Likely as not, that structure is embedded within a larger structure containing the information the callback needs anyway, so a simple `container_of()` call can replace the casting of the `unsigned long` value. 

As might be expected, though, Cook has concerns about this API that go beyond matching the current kernel style. One of those is that a buffer overflow in the area of a `timer_list` structure may be able to overwrite both the function pointer and the data passed to the called function, allowing arbitrary calls within the kernel. That, naturally, makes `timer_list` structures interesting to attackers, and explains why Cook has been [trying to harden timers](<https://lwn.net/Articles/731082/>) for a while. The prototype of the timer callback, containing a single `unsigned long` argument, is also evidently an impediment to ""future control flow integrity work"". It would be better if the callback had a unique prototype that was visibly different from all of the other kernel functions taking an `unsigned long` argument. 

Cook has been working on changes to the timer interface for a while in an attempt to address these issues. The core idea is simple: get rid of the `data` value and just pass the `timer_list` structure to the timeout function. The actual transition, though, is complicated by the existence of 800 or so `setup_timer()` call sites in the kernel now. Trying to change them all at once would not be anybody's idea of fun, so a phased approach is needed. 

In this case, Cook has introduced a new function for the initialization of timers: 

```
void timer_setup(struct timer_list *timer, void (*callback)(struct timer_list *),
    		     unsigned int flags);
```

For the time being, `timer_setup()` simply stores a pointer to `timer` in the `data` field. Note that the prototype of the callback has changed to expect the `timer_list` pointer. 

With that function in place, calls to `setup_timer()` can be replaced at leisure, as long as each corresponding timer callback function is adjusted accordingly. For the most part, as can be seen in [this example](<https://lwn.net/Articles/735892/>), the changes are trivial. Many timer callbacks already were casting the `data` value to a pointer to the structure they needed; they just need a one-line change to obtain that from the `timer_list` pointer instead. A new `from_timer()` macro has been added to make those conversions a bit less verbose. 

The addition of `timer_setup()` was merged just prior to the 4.14-rc3 release — rather later in the release cycle than one would ordinarily expect to see the addition of new interfaces. The purpose of this timing was clear enough: it clears the way for the conversion of all of those `setup_timer()` calls, a task which, it is hoped, will be completed for the 4.15 kernel release. Once that is done, the underlying implementation can be changed to drop the `data` value and the `setup_timer()` interface can be removed entirely. At the end, the kernel will be equipped with a timer mechanism that is a little bit more efficient, more like other in-kernel APIs, and easier to secure.
