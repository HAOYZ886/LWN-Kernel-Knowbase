---
title: "Inter-event tracing"
url: https://lwn.net/Articles/714511/
date: "February 15, 2017"
category: Tracing
author: "By Jonathan Corbet February 15, 2017"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
February 15, 2017

The kernel's tracing infrastructure is primarily concerned with events, which are usually tied to the execution of specific blocks of code. But the interesting information is often in what happens _between_ events. The most obvious variable of this type that one might want to monitor is timing — how much time elapses between one event and another that follows from it? — but others exist as well. Options for the calculation of inter-event values include the use of BPF programs or postprocessing in user space; a new patch set may soon add the ability to perform these calculations directly in the kernel instead. 

Tom Zanussi's [inter-event support patch set](<https://lwn.net/Articles/714001/>) is clearly focused on timing measurements. It is an extension of his [histogram triggers work](<https://lwn.net/Articles/635522/>) that was merged in the 4.6 development cycle. That work provides a mechanism for the storage of data from events, but it can only do one thing with the result: generate a histogram. This storage capability has the potential for other uses and can be employed for inter-event tracing, but there are a few problems that need to be solved for that to be possible. 

The first of those is to arrange for the storage of data from one event to be used later when another event fires. An example provided with the patch set involves the `sched_wakeup` event, which fires when the kernel decides that a sleeping process is now runnable and should wake up. Consider the following command: 

```
echo hist:keys=pid:ts0=common_timestamp.usecs \
             >> /sys/kernel/debug/tracing/events/sched/sched_wakeup/trigger
```

This command establishes a special sort of "histogram" (really just using the histogram mechanism's data-storage capability) on the `sched_wakeup` event. The `keys=pid` directive uses the ID of the process to be awakened as the key with which the data is stored. The actual data associated with that key is specified with `ts0=common_timestamp.usecs`. It creates a new variable, `ts0`, that remembers the current time when the event was fired. The `common_timestamp` field is also new; it makes timestamp information available on any event. 

The above records when the kernel decided to wake a process; now it is time to compute how long it takes until that process actually runs in a CPU. That time is the wakeup latency experienced by the process; one generally wants it to be as low as possible and, in a realtime setting, it must not exceed the maximum the system can tolerate. The wakeup latency can be calculated using the `sched_switch` event, which fires when a new process is given access to the processor. That is done with a command like: 

```
echo 'hist:keys=woken_pid=next_pid:woken_prio=next_prio:\
              wakeup_lat=common_timestamp.usecs-ts0:\
    	  onmatch().trace(wakeup_latency)' \
              >> /sys/kernel/debug/tracing/events/sched/sched_switch/trigger
```

There are a few things happening here, needless to say. The `keys=woken_pid=next_pid` fragment takes the `next_pid` event variable (identifying the process to be switched to in the processor), assigns it to a new variable called `woken_pid`, and uses it as the key into the histogram data. The next fragment, `woken_prio=next_prio`, stores the priority of the new process in the new `woken_prio` variable. Things get a bit more complicated with: 

```
wakeup_lat=common_timestamp.usecs-ts0
```

Here, the `ts0` timestamp value that was saved in the `sched_wakeup` event is recalled and subtracted from the current time, yielding the latency; that value is then stored in yet another new variable called `wakeup_lat`. 

The `onmatch()` directive in the above command relates to how the computed latency is reported. That value is computed from two separate events and, thus, is not really associated with either of the events described above, so it should not be reported with them. Instead, the patch set creates a new abstraction called a "synthetic event" that is used to report calculated, inter-event values. In this case, such an event can be created with a command like: 

```
echo 'wakeup_latency lat=sched_switch:wakeup_lat \
                             pid=sched_switch:woken_pid \
                             prio=sched_switch:woken_prio' \
             >> /sys/kernel/debug/tracing/synthetic_events
```

This command creates a new event called `wakeup_latency`; three variables are created, essentially as pointers to variables in the `sched_switch` event. One of those, `wakeup_lat` (redefined as `lat` here) is the calculated latency. 

With that event in place, we can look at the final part of the previous command: 

```
onmatch().trace(wakeup_latency)
```

The `onmatch()` pseudo-function acts if there is a match on the histogram key (the process ID in this case); when it acts, it will cause the synthetic event named in the `trace()` "call" to be fired. That event behaves like any other event in all respects; it can be read out to user space or used to create a histogram, for example. 

With the above commands, it is possible to monitor wakeup latencies in the system. The command set has been simplified a bit from the original commands in the patch set, which include extra filtering to limit latency tracking to processes running the `cyclictest` testing tool. For more details, see the patch set announcement linked above or [the documentation](<https://lwn.net/Articles/714516/>) that is part of the patch set itself. 

The patch set has seen a fair amount of active review, resulting in generally positive comments. It does seem likely, though, that the details of the syntax as described above will change before this work is considered ready for merging. Nobody has, so far, suggested that attached BPF programs should be used to perform these calculations, though the problem could conceivably be solved that way. In any case, for now, the patch set is being reworked to reflect the review comments received so far; a new version should be expected before too long.
