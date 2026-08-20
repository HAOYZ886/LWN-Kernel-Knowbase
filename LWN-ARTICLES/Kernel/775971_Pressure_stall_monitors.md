---
title: Pressure stall monitors
url: https://lwn.net/Articles/775971/
date: "January 4, 2019"
category: Performance monitoring
author: "By Jonathan Corbet January 4, 2019"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
January 4, 2019

One of the useful features added during the 4.20 development cycle was the availability of [pressure-stall information](<https://lwn.net/Articles/759781/>), which provides visibility into how resource-constrained the system is. Interest in using this information has spread beyond the data-center environment where it was first implemented, but it turns out that there some shortcomings in the current interface that affect other use cases. Suren Baghdasaryan has posted [a patch set](<https://lwn.net/ml/linux-kernel/20181214171508.7791-1-surenb@google.com/>) aimed at making pressure-stall information more useful for the Android use case — and, most likely, for many other use cases as well. 

As a reminder, the idea behind the pressure-stall mechanism is to track the amount of time that processes are unable to execute because they are waiting for resources (for CPU time, memory, and I/O bandwidth in particular). For example, reading `/proc/pressure/memory` will yield output like: 

```
some avg10=70.24 avg60=68.52 avg300=69.91 total=3559632828
        full avg10=57.59 avg60=58.06 avg300=60.38 total=3300487258
```

This output says that at least one process has been blocked waiting for memory 70.24% of the time over the last ten seconds, or 68.52% of the time over the last minute. In the last ten seconds, _all_ processes have been stalled 57.59% of the time, indicating a system that is seriously short of memory. An orchestration system monitoring this system would see that over half the CPU time is going to waste because the demands on memory are too high; corrective action is probably indicated. 

The Android runtime system also tries to manage the set of running processes to make the best use of the hardware while providing acceptable response times to the user. When memory gets tight, for example, background processes may be killed to ensure that the application the user is engaging with at the moment has the resources it needs to run quickly. The pressure-stall information has some obvious utility when it comes to this kind of automated resource management: it provides exactly the kind of information needed to determine whether the system's response time is being affected by a shortage of memory. 

The problem, from the Android point of view, is that the information provided is too little and too late. The highest-resolution information available is aggregated over ten seconds; that is entirely adequate for most data-center settings, but it's far too slow for a device that is interacting directly with users. If it takes ten seconds to learn that the device is getting sluggish, the user is likely to be getting grumpy by the time any corrective action is taken. Such users might well conclude that they are better off not staring into their phone all day, and that would clearly be bad for the industry as a whole. 

The answer to this problem is to extend the pressure-stall mechanism to allow for high-frequency monitoring of stall data. With the patch set applied, an interested application can open `/proc/pressure/memory` for write access, then write a line containing three pieces of information: 

```
type stall-trigger time-window
```

The `type` value is either `some` (indicating that information about any stalled process is wanted) or `full` (limiting the information to full-system stalls where no process can run). `stall-trigger` indicates (in microseconds) the stall time that will trigger an event, and `time-window` is the time period over which that stall time happens. So, for example, writing: 

```
full 100000 1000000
```

will cause the monitor to trigger when the system stalls for a minimum of 100ms over any 1s period. The minimum `time-window` is 500ms, while the maximum is 10s. The `stall-trigger` can also be expressed as a percentage value; "`10%`" asks for a stall time that is 10% of the given time window. 

Having requested a stall notification, the application can then pass the file descriptor to [`poll()`](<http://man7.org/linux/man-pages/man2/poll.2.html>). An exceptional condition (`POLLPRI`) event will be returned whenever a notification is generated. A monitoring system can thus be notified within a half-second of the system starting to become unresponsive and act to address the situation. There can be multiple processes monitoring the same stall information with different triggers and time windows. As is the case with the current pressure stall information, the new mechanism is aware of control groups; opening the relevant files within a memory control-group hierarchy will provide information on the members of that group only. 

The actual tracking of stall times has been kept simple to avoid adding to the load on the system. For each monitor, the accumulated stall time is checked ten times for each time window. If the current window is 50% past, the calculated stall value will be the time accumulated so far in this window, plus 50% of the total from the previous window. This mechanism assumes that the situation will not change hugely from one window to the next; the benefit is that it only has to store a single past value for each monitor. The monitoring is turned off entirely if no stall events are occurring, so its overhead should be zero on a lightly loaded system. 

The end result, Baghdasaryan says, is good: 

With these patches applied, Android can monitor for, and ward off, mounting memory shortages before they cause problems for the user. For example, using memory stall monitors in userspace low memory killer daemon (lmkd) we can detect mounting pressure and kill less important processes before [the] device becomes visibly sluggish. 

The functionality provided by this patch set seems clearly worthwhile, but the code itself is going to need a bit of work yet. The biggest [complaint](<https://lwn.net/ml/linux-kernel/20181217145754.GB2218@hirez.programming.kicks-ass.net/>) came from Peter Zijlstra, who doesn't like the elimination of the "idle mode" that stops data collection entirely when little is going on. Keeping the collection running will prevent the system from going into its deepest idle states, which will not be good for power consumption. Some sort of solution to that problem will need to be found before this code can go upstream. 

There were also some comments on the string-parsing code added by the patch set; it may be simplified by eliminating the percentage option described above. Beyond that, it seems clear that this is a welcome addition to the system's load-monitoring functionality. Chances are it will find its way upstream before too long. How long it will be stalled before finding its way into production handsets is rather less clear, of course.
