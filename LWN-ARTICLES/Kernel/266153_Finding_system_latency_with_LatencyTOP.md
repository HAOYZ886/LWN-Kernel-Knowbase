---
title: Finding system latency with LatencyTOP
url: https://lwn.net/Articles/266153/
date: "January 23, 2008"
category: Latency
author: "By Jake Edge January 23, 2008"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jake Edge**  
January 23, 2008

Stuttering audio or an unresponsive desktop - typically caused by operating system latency - are two things that annoy users. They can be difficult problems to diagnose, though, as they are transient and buried deep inside the kernel. A new tool, [LatencyTOP](<http://www.latencytop.org/>), seeks to provide more information on where latency is occurring so that it can be fixed or avoided. 

Latency is the measure of how much time elapses between when an action is initiated and when its effects become visible. If a user clicks the mouse button in an application, the latency is the amount of time between that click and when the associated action begins. There are lots of different reasons for latency, some of which are outside of Linux's control; being able to measure what latency the OS is contributing will be very useful. LatencyTOP is reporting on a specific subset of latency causes, as described in the [announcement](<http://lwn.net/Articles/265945/>): 

There are many types and causes of latency, and LatencyTOP [focuses on the] type that causes audio skipping and desktop stutters. Specifically, LatencyTOP focuses on the cases where the applications want to run and execute useful code, but there's some resource that's not currently available (and the kernel then blocks the process). This is done both on a system level and on a per process level, so that you can see what's happening to the system, and which process is suffering and/or causing the delays. 

LatencyTOP measures the average and maximum amount of latency in various operations by inserting annotation calls in the kernel. An example from the announcement is instructive: 

```
asmlinkage long sys_sync(void)
         {
        +       struct latency_entry reason;
        +       set_latency_reason("sync system call", &reason);
                do_sync(1);
        +       restore_latency_reason(&reason);
        +
                return 0;
         }
```

The scheduler accumulates any time spent sleeping, between the `set_latency_reason()` and `restore_latency_reason()` calls, charging it to the "sync system call". Any lower level calls to set the latency reason will be ignored in this code path - they may be useful in other code paths - as it is the highest level active reason that gets charged. 

The current interface for annotating is likely to change, though the semantics will stay the same. Comments on the original submission suggested using the [kernel markers](<http://lwn.net/Articles/245671/>) feature that was merged for 2.6.24. LatencyTOP developer Arjan van de Ven seems amenable to that; reusing a kernel interface, rather than adding a new one, is generally the right choice. There is other work to do as well, the patch was submitted for other kernel hackers to test and comment on, not to be merged into the mainline. 

[![\[LatencyTOP
application\]](https://static.lwn.net/images/latencytop_sm.png)](<http://lwn.net/Articles/266279/>)

LatencyTOP comes with a userspace application, shown at right, that displays the information gathered. It reads from the `/proc/latency_stats` file that is created by the [LatencyTOP infrastructure patch](<http://lwn.net/Articles/266246/>) - so long as you enable CONFIG_LATENCYTOP in the kernel. It displays the nine - an off-by-one in the code as it would seem that ten were intended - largest latencies over the past 30 seconds in the upper pane. 

[![\[LatencyTOP
application\]](https://static.lwn.net/images/latencytop2_sm.png)](<http://lwn.net/Articles/266291/>)

A list of process names runs along the bottom of the display, which can be selected with the arrow keys. The latency sources for that process will then be shown in the lower pane. The example at left shows the tool with the firefox process selected. As can be seen, there are still lots of areas that need annotations - "Unknown reason" along with the wait channel are displayed when the reason has not been set. When narrowing a problem down, it should be straightforward for a kernel hacker to add annotations to the appropriate locations. 

LatencyTOP, like its sibling [PowerTOP](<http://www.lesswatts.org/projects/powertop/>) - also developed by van de Ven at the Intel Open Source Technology Center - is a powerful tool for trying to track down system problems. It will probably undergo some changes along the way: the userspace application is still rather rudimentary and the kernel data collection needs finer-grained locking. But, before too long, a mainstream tool to measure system latency based on this work should appear.
