---
title: Better CPU selection for timer expiration
url: https://lwn.net/Articles/913568/
date: "November 7, 2022"
category: "Releases-6.9; Timers"
author: "By Jonathan Corbet November 7, 2022"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
November 7, 2022

On the surface, the kernel's internal timer mechanism would not appear to have changed much in a long time; the core API looks quite similar to the one present in the 1.0 release. Underneath the API, naturally, quite a bit of complexity has been added over the years. The implementation of this API looks to become even more complex — but faster — if and when [this patch set](<https://lwn.net/ml/linux-kernel/20221104145737.71236-1-anna-maria@linutronix.de/>) from Anna-Maria Behnsen finds its way into the mainline. 

#### Premature optimization

For context, it is worth remembering that the kernel actually has two core APIs for the management of internal timers. [High-resolution timers](<https://docs.kernel.org/next/timers/hrtimers.html>) (hrtimers) are, as their name would suggest, used for near-future events where precision is important; they are relatively expensive and only used when necessary. For everything else, there is the subsystem just known as "kernel timers" (or, sometimes, "timer-wheel timers"), centered around functions like [`add_timer()`](<https://docs.kernel.org/next/driver-api/basics.html?#c.add_timer>). Behnsen's patch set changes how these ordinary kernel timers work. 

Arguably, most uses of kernel timers are the kernel ensuring that it will respond properly if an expected event doesn't happen. A driver may start an I/O operation on a device confident in the knowledge that the device will raise an interrupt when the operation completes, but it will still set a timer so that, if the device goes out to lunch, the operation doesn't just hang indefinitely. Other parts of the kernel, such as networking, use timers in similar ways. 

There are a couple of interesting implications that arise from this usage pattern. One is that such timers need not expire at exactly their nominal expiration time; if the kernel takes a little while to get around to handling an expired timer, nothing bad happens. That allows the kernel to batch timer expirations and to defer them to avoid waking an otherwise idle CPU. The implication that is relevant here, though, is that kernel timers rarely expire. When things are operating normally, the expected events will occur and the timer will either be canceled or reset to a new time further in the future. As a result, the timer subsystem should be optimized for the creation and cancellation of timer events. And, indeed, much effort has gone into that sort of optimization, as can be seen in LWN coverage from as far back as [2004](<https://lwn.net/Articles/84836/>) up to [a significant reimplementation](<https://lwn.net/Articles/646950/>) of timers in 2015\. 

Behnsen has identified a place where further optimization can be performed, though. When a timer is set in current kernels, the timer subsystem spends some CPU time trying to decide which CPU should handle the expiration of that timer. The intent is to push that work to a CPU that is already busy rather than waking an idle CPU just to expire a timer. So the timer subsystem scans through the system looking for a suitable CPU and adds the timer to the appropriate queue. 

There are a couple of problems with this algorithm, though. One is that a CPU that is busy now may no longer be busy when the timer expires. So the choice of a CPU to handle expiration, if made when the timer is set, is really just a guess. Perhaps worse, though, is that this work is being done at the wrong time; since most timers never expire, any effort that is put into picking the expiration CPU ahead of time is likely to be wasted, even if the guess turns out to be a good one. It would be far better to not do any extra work when a timer is set, and have a CPU that is actually busy at expiration time take care of it then — on the relatively rare occasion when a timer actually expires. 

The first part — not picking an expiration CPU at setting time — is easy to implement; a timer is just put into the queue of the CPU that is setting it. Having a suitable CPU actually handle expiration is harder, though. A naive implementation might just create a simple queue of timers that a CPU would check occasionally if it's running and able to handle expirations. That would create a great deal of locking contention and cache-line bouncing, though, slowing things down even when there were no timers to handle. So something more complex is called for. 

#### Choosing the expiration CPU

The scheme chosen is to organize the system's CPUs into a hierarchy that resembles the hardware topology, but which is independent from it. At the lowest level, CPUs are assembled into groups of up to eight, with the constraint that all eight must be contained within the same NUMA node. The groups are, themselves, organized into groups; this process continues until all CPUs in the system have been arranged into a single tree. 

![\[CPU-group organization\]](https://static.lwn.net/images/2022/timer-exp.svg) Consider, for example, a simple, four-CPU system organized into two NUMA nodes as shown to the right. The first two CPUs are organized into Group 1; the other two, since they are in a different NUMA node, go into a separate group. Those two groups, in turn, are placed together in Group 3\. A larger and more complex system might require more levels of group hierarchy, but that gets awkward to show in a simple diagram. 

The timer API allows timers to be pinned to specific CPUs; that does not change in the reimplementation. Each CPU will have to handle expiration for its pinned timers, even if that means waking from an idle state. Most timers, though, can be executed anywhere in the system and are not pinned; the handling of these "global" timers will be different from before. A CPU that is busy will continue to handle global timers that are in its specific queue but, if that CPU goes idle, it will instead add its soonest-expiring global timer to the queue associated with its group. 

Normally, each CPU group will designate one of its members as the "migrator". That CPU, which cannot be idle, will occasionally check the queue associated with its group for expiring global timers; if the CPU that queued a timer there is still idle, then the migrator will pull it over and handle the expiration instead, and the CPU that initially queued the timer can remain idle. So, for example, if CPU 1 in the diagram above is idle, it will have enqueued its earliest-expiring global timer in Group 1; if CPU 2 is running (and is thus the migrator), it will handle that timer when it expires. 

If the migrator goes idle, then another CPU in the group has to be handed the baton and become the new migrator; that is just a matter of finding the next busy CPU in that group. If all of the other CPUs are also idle, instead, then the group ends up without a migrator. In this case, the group is marked as idle in the _next_ higher group in the hierarchy, and its first-expiring timer is queued at that next level. So, if CPU 2 also goes idle, it will take the earliest-expiring event in Group 1 and put it into the queue at Group 3\. 

The assignment of the migrator role happens in the higher-level groups as well. If a group contains other groups, one of those groups will be the migrator for that level. In the scenario described here, Group 2 will be the migrator for Group 3\. The CPU running as the migrator within Group 2 (either CPU 3 or CPU 4) will thus have to handle timer events for Group 3 as well. In a system with a lot of idle CPUs, this migrator role can propagate all the way to the top of the hierarchy, at which point one CPU may be responsible for handling all expiring timers in the system. 

If _that_ CPU also goes idle, the system will be left without any migrator CPU at all. At that point, the last CPU standing will set a hardware timer to wake it at the expiration time for the earliest expiring timer. That ensures that timer events don't get dropped even in the absence of busy CPUs to handle them. 

Implementing this machinery in the timer subsystem results in a patch set adding nearly 2,000 lines of code to a core kernel subsystem. The benefit that comes from this work is said to be an approximately 25% reduction in the time required to add and remove a timer. Since timers can be set (and changed) within performance-sensitive code, this improvement likely justifies the added complexity. 

A side benefit of this work is that it should enable the removal of the [deferrable timers](<https://lwn.net/Articles/228143/>) mechanism. Deferrable timers are those for which expiration can be deferred without any ill effect; a CPU that is going idle will not wake up solely to handle a deferrable timer. Turning deferrable timers into global timers will have the same effect — they will no longer cause a sleeping CPU to wake — so there is no longer a need to handle them separately. The removal of deferrable timers, which is, according to Behnsen, coming soon, will counterbalance some of the complexity added by this work.
