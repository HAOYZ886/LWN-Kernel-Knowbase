---
title: Software interrupts and realtime
url: https://lwn.net/Articles/520076/
date: "October 17, 2012"
category: "Interrupts-Software; Realtime"
author: "By Jonathan Corbet October 17, 2012"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
October 17, 2012

The Linux kernel's software interrupt ("softirq") mechanism is a bit of a strange beast. It is an obscure holdover from the earliest days of Linux and a mechanism that few kernel developers ever deal with directly. Yet it is at the core of much of the kernel's most important processing. Occasionally softirqs make their presence known in undesired ways; it is not surprising that the kernel's frequent problem child — the realtime preemption patch set — has often run afoul of them. Recent versions of that patch set embody a new approach to the software interrupt problem that merits a look. 

#### A softirq introduction

In the announcement for the [3.6.1-rt1](<https://lwn.net/Articles/518993/>) patch set, Thomas Gleixner described software interrupts this way: 

First of all, it's a conglomerate of mostly unrelated jobs, which run in the context of a randomly chosen victim w/o the ability to put any control on them. 

The softirq mechanism is meant to handle processing that is almost — but not quite — as important as the handling of hardware interrupts. Softirqs run at a high priority (though with an interesting exception, described below), but with hardware interrupts enabled. They thus will normally preempt any work except the response to a "real" hardware interrupt. 

Once upon a time, there were 32 hardwired software interrupt vectors, one assigned to each device driver or related task. Drivers have, for the most part, been detached from software interrupts for a long time — they still use softirqs, but that access has been laundered through intermediate APIs like tasklets and timers. In current kernels there are ten softirq vectors defined; two for tasklet processing, two for networking, two for the block layer, two for timers, and one each for the scheduler and read-copy-update processing. The kernel maintains a per-CPU bitmask indicating which softirqs need processing at any given time. So, for example, when a kernel subsystem calls `tasklet_schedule()`, the `TASKLET_SOFTIRQ` bit is set on the corresponding CPU and, when softirqs are processed, the tasklet will be run. 

There are two places where software interrupts can "fire" and preempt the current thread. One of them is at the end of the processing for a hardware interrupt; it is common for interrupt handlers to raise softirqs, so it makes sense (for latency and optimal cache use) to process them as soon as hardware interrupts can be re-enabled. The other possibility is anytime that kernel code re-enables softirq processing (via a call to functions like `local_bh_enable()` or `spin_unlock_bh()`). The end result is that the accumulated softirq work (which can be substantial) is executed in the context of whichever process happens to be running at the wrong time; that is the "randomly chosen victim" aspect that Thomas was talking about. 

Readers who have looked at the process mix on their systems may be wondering where the `ksoftirqd` processes fit into the picture. These processes exist to offload softirq processing when the load gets too heavy. If the regular, inline softirq processing code loops ten times and still finds more softirqs to process (because they continue to be raised), it will wake the appropriate `ksoftirqd` process (there is one per CPU) and exit; that process will eventually be scheduled and pick up running softirq handlers. `Ksoftirqd` will also be poked if a softirq is raised outside of (hardware or software) interrupt context; that is necessary because, otherwise, an arbitrary amount of time might pass before softirqs are processed again. In older kernels, the `ksoftirqd` processes ran at the lowest possible priority, meaning that softirq processing was, depending on where it is being run, either the highest priority or the lowest priority work on the system. Since 2.6.23, `ksoftirqd` runs at normal user-level priority by default. 

#### Softirqs in the realtime setting

On normal systems, the softirq mechanism works well enough that there has not been much motivation to change it, though, as described in "[The new visibility of RCU processing](<https://lwn.net/Articles/518953/>)," read-copy-update work has been moved into its own helper threads for the 3.7 kernel. In the realtime world, though, the concept of forcing arbitrary processes to do random work tends to be unpopular, so the realtime patches have traditionally pushed all softirq processing into separate threads, each with its own priority. That allowed, for example, the priority of network softirq handling to be raised on systems where networking needed realtime response; conversely, it could be lowered on systems where response to network events was less critical. 

Starting with the 3.0 realtime patch set, though, that capability went away. It worked less well with the [new approach to per-CPU data](<https://lwn.net/Articles/452884/>) adopted then, and, as Thomas said, the per-softirq threads posed configuration problems: 

It's extremely hard to get the parameters right for a RT system in general. Adding something which is obscure as soft interrupts to the system designers todo list is a bad idea. 

So, in 3.0, softirq handling looked very similar to how things are done in the mainline kernel. That improved the code and increased performance on untuned systems (by eliminating the context switch to the softirq thread), but took away the ability to finely tweak things for those who were inclined to do so. And realtime developers tend to be highly inclined to do just that. The result, naturally, is that some users complained about the changes. 

In response, in 3.6.1-rt1, the handling of softirqs has changed again. Now, when a thread raises a softirq, the specific interrupt in question (network receive processing, say) is remembered by the kernel. As soon as the thread exits the context where software interrupts are disabled, that one softirq (and no others) will be run. That has the effect of minimizing softirq latency (since softirqs are run as soon as possible); just as importantly, it also ties processing of softirqs to the processes that generate them. A process raising networking softirqs will not be bogged down processing some other process's timers. That keeps the work local, avoids nondeterministic behavior caused by running another process's softirqs, and causes softirq processing to naturally run with the priority of the process creating the work in the first place. 

There is an exception, of course: softirqs raised in hardware interrupt context cannot be handled in this way. There is no general way to associate a hardware interrupt with a specific thread, so it is not possible to force the responsible thread to do the necessary processing. The answer in this case is to just hand those softirqs to the `ksoftirqd` process and be done with it. 

A logical next step, hinted at by Thomas, is to move from an environment where all softirqs are disabled to one where only specific softirqs are. Most code that disables softirq handling is only concerned with one specific handler; all the others could be allowed to run as usual. Going further, he adds: ""the nicest solution would be to get rid of them completely."" The elimination of the softirq mechanism has been on the "todo" list for a long time, but nobody has, yet, felt the pain strongly enough to actually do that work. 

The nature of the realtime patch set has often been that its users feel the pain of mainline kernel shortcomings before the rest of us do. That has caused a great many mainline fixes and improvements to come from the realtime community. Perhaps that will eventually happen again for softirqs. For the time being, though, realtime users have an improved softirq mechanism that should give the desired results without the need for difficult low-level tuning. Naturally, Thomas is looking for people to test this change and report back on how well it works with their workloads.
