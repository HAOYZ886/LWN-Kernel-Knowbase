---
title: Scheduling for Android devices
url: https://lwn.net/Articles/706374/
date: "November 15, 2016"
category: "Android; Scheduler-and power management"
author: "By Jonathan Corbet November 15, 2016 Linux Plumbers Conference"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
November 15, 2016

* * *

[Linux Plumbers Conference](<https://lwn.net/Archives/ConferenceByYear/#2016-Linux_Plumbers_Conference>)

The [Android/Mobile microconference](<https://www.linuxplumbersconf.org/2016/ocw/events/LPC2016/tracks/579>) at the [2016 Linux Plumbers Conference](<https://www.linuxplumbersconf.org/2016/>) was a grueling seven-hour affair. Your editor was unable to attend the entire session, but was fortunate enough to be present for a series of talks on scheduling. Adding better awareness of power usage to the scheduler has been a recurring topic for some years; that didn't change this year, but there was also a focus on improving the user experience in general. 

Todd Kjos started by talking about scheduling for the Nexus 5X handset, which was the first of Google's devices to be based on the big.LITTLE processor architecture. This handset has four A53 and two A57 cores in it; the A53s are relatively slow and power-efficient, while the A57s offer higher performance. Tuning Android to run well on this device has turned out to be challenging. The objectives are straightforward enough: workloads that the user doesn't care about should run on the A53 cores, but performance-sensitive loads should not get stuck on those cores. When not much is being done, the A57 cores should not be running at all. But achieving those objectives has taken some work. 

It would be nice to avoid repeating that work for each new device, but generic support for energy-aware scheduling still has not found its way into the mainline kernel. Instead, each system-on-chip (SoC) vendor provides its own heavily patched kernel with its own solution to the [![\[Todd
Kjos\]](https://static.lwn.net/images/conf/2016/lpc/ToddKjos-sm.jpg)](<https://lwn.net/Articles/706412/>) scheduling problem. Some of these vendors, Kjos said, have done a relatively good job with their scheduling patches; others less so. But, either way, these out-of-tree patches mean that vendors have a lot of work to do when new Android releases come along. 

All of these solutions are hampered by a lack of knowledge about what is going on — and what the user most cares about — in user space. But the Android framework does have an idea about what is going on there. At any given time, Android knows what the "top app", the app that the user actually sees and interacts with, is. It is aware of other "foreground" apps that are a part of the user experience; they may be drawing other parts of the screen or occupying a part of a split screen. Then, there are the "background" processes that are not currently a part of the user's experience and are, thus, less important. 

Beyond that, the Android framework is aware of user interaction events, such as touches and swipes, and some other aspects of the workload, including things like video frame rates. That is a lot of information about what is going on, but that information is not being exploited on most devices currently. 

The scheduler on an Android device should provide the best experience for the user while being as energy-efficient as possible. The latter goal implies getting proper energy-aware scheduling support into the mainline kernel. A common solution would allow Android to perform workload-specific tuning in a generic manner and would reduce the amount of SoC-specific code carried by manufacturers and vendors. As CPU topologies become more complex, the need to provide a better, saner strategy grows. 

Energy-aware scheduling, as presented by Kjos, consists of three major components. The first of those is the core scheduler changes allowing it to use CPU topology to place running processes optimally. Some of that work has found its way into the mainline, and progress is being made on the rest. Next is improved frequency and voltage selection for the running processors to maximize efficiency; the plan here is to switch over to the mainline [schedutil](<https://lwn.net/Articles/682391/>) framework, which can make better use of what the scheduler knows to configure the CPU operating parameters. The last piece is the out-of-tree SchedTune subsystem, which allows Android to tweak scheduling policies on the fly. 

#### A digression into SchedTune

Your editor will now indulge in a bit of temporal manipulation. Patrick Bellasi presented the SchedTune governor later that day, but the information provided then is useful for the understanding of the rest of Kjos's talk. In the interest of a better narrative, we'll take a quick look at SchedTune now. 

The kernel's current CPU-frequency governors attempt to match the CPU's operating power points (its frequency and voltage) to the workload that is actually being run. A light workload can be run at a relatively low frequency and voltage, saving power, but heavier workloads require the CPU to be running in a more energy-intensive mode. The optimal power point, he said, should be chosen with the user's needs in mind; sometimes response time is crucial to the user's experience, while, at other times, workloads can be run more slowly and nobody will notice. 

Unfortunately, the scheduler knows little about which processes are important to the user at any given time. On Android systems, the runtime system does have a good idea of what matters, but there is no good way to [![\[Patrick Bellasi\]](https://static.lwn.net/images/conf/2016/lpc/PatrickBellasi-sm.jpg)](<https://lwn.net/Articles/706413/>) communicate that information to the scheduler. The [SchedTune governor](<https://lwn.net/Articles/704859/>) is meant to be a way to provide that information so that the scheduler can act on it. 

In current kernels, the load-tracking subsystem maintains an estimate of just how much load each running process will put onto the system. Recent changes have enabled CPU-frequency governors to make use of that information to set the power points optimally; usually, the objective is to run the CPUs just fast enough to get the anticipated amount of work done. Governor policy can be changed into, for example, a "performance" mode where the objective is to get the work done as quickly as possible, but there is no way to set CPU-frequency policies on a per-task basis or, in other words, to tell the system that it is worthwhile to expend a certain amount of extra energy to get a specific job done. 

SchedTune is implemented as a control-group controller. Each control group has a single tunable knob, called `schedtune.boost`, which can be used by the runtime system to change how processes within that group are scheduled. This parameter works in an interesting manner: it tweaks the load-tracking code to make the affected processes look heavier (or lighter) than they really are. If a group is boosted by 25%, the scheduler will expect it to use 25% more CPU time than it (probably) actually will, and the CPU-frequency governor will speed up the processor accordingly. "Boosting" a process in this way, thus, does not affect its scheduling priority, but it will affect the speed of the CPU on which it ends up running. 

Bellasi concluded with some proposals for future enhancements to SchedTune. One of those would be to tweak scheduling further so that processes with a positive boost value would get longer time slices in the CPU. A couple more knobs may be added to the control group to affect the CPUs on which processes may be run; they would specify the minimum and maximum performance capacity that a candidate CPU may have. To summarize, while SchedTune is already in use in shipping products, it is still in a certain amount of flux. 

#### Energy-aware scheduling in the Pixel phone

Returning to Kjos's talk: with these components in mind, he and others set out to implement proper energy-aware scheduling in the Pixel phone; the initiative began after a meeting at the 2015 Linux Plumbers Conference. Some experiments with energy-aware scheduling on a tablet device yielded good results, so a group of engineers from ARM, Qualcomm, and Google decided to collaborate on a solution for the Pixel. The group had a significant challenge: come up with a relatively generic energy-aware scheduling solution that worked at least as well as Qualcomm's out-of-tree QHMP scheduling patches. 

Getting there required a number of modifications to the existing energy-aware scheduling code. Its view of big.LITTLE processors doesn't quite fit the Snapdragon 821 used in the Pixel, so some of those assumptions had to be changed. SchedTune and cpusets were adopted to allow the Android framework to specify how processes should be placed in the system and, in particular, whether specific processes should be spread out to idle CPUs or packed into a small number of CPUs. SchedTune was also enhanced to allow tasks to be marked as latency-sensitive. 

The kernel's [per-entity load tracking](<https://lwn.net/Articles/531853/>) mechanism, by which the scheduler tracks how much load each process puts on the system, proved to not be up to the task of getting the best performance out of this processor, so the out-of-tree [window-assisted load tracking](<https://lwn.net/Articles/704903/>) (WALT) mechanism was used instead. Nobody discussed the WALT patches in detail during the microconference, but they do play a significant part in this work, and thus merit a quick discussion. 

#### Digression 2: WALT

Per-entity load tracking was added to the 3.8 kernel to fill in a gap in the scheduler's understanding: it didn't have any way to know how much load any given process would put on the CPU. That information is useful for load balancing — distributing processes optimally across the CPUs in the system — and for knowing what will happen if a process is moved from one CPU to the next. Each process's load is calculated using a simple geometric series, with the most recent load being counted most heavily. 

This mechanism is an improvement over what came before, but it turns out to not work ideally, especially in the mobile world. The biggest complaint seems to be that it is simply too slow to respond to changes in a process's behavior. A web browser that has been idle for a long time, with a correspondingly low calculated load, may suddenly find itself rendering a complex page and needing a lot of CPU. If its associated load does not rise quickly enough, that browser process will not get enough CPU time (because it must share a CPU with too many others, putting the system out of balance, or because the CPU frequency will not be increased as needed) while it is trying to do its work. The result is slow response as seen by the increasingly grumpy user. 

The problem also exists when processes stop using the CPU. Per-entity load tracking will take some time to notice that the load is gone, with the result that the CPU will stay in a higher power state than it actually needs to be, shortening battery life. To make things worse, processes that are not runnable (waiting on I/O, for example) are counted as part of the load, even though they are not doing anything and may not for some time. 

The WALT patches do away with the geometric series for load calculations. Instead, WALT simply tracks the amount of CPU time used by the process during a set of recent "windows" of time. According to the patch description, "N" windows are maintained, and a process's calculated load is the maximum of either the most recently completed window or the average of all N windows. The actual patch, as posted, appears to keep around exactly one 20ms window, so the most recent measurement and the average are one and the same. In essence, WALT has modified per-entity load tracking by chopping off all but the first term in the geometric load-calculation series. There is one other significant change: tracking is only done while the process is running; a process will not contribute to system load while it is waiting for something, and its previous load calculation will still be there once it becomes runnable again. 

This algorithm causes WALT to respond much more quickly to a process's behavior; if a process begins some sort of CPU-intensive activity, its calculated load will quickly rise to match. It is thus useful for tasks like setting the CPU frequency. It may not work as well as per-entity load tracking for the long-term balancing of processes across the system; for this reason, the latter numbers are still used for load-balancing decisions. 

The WALT patches got [a bit of a grumpy reception](<https://lwn.net/Articles/706409/>) from scheduler maintainer Peter Zijlstra, who would have liked to have seen them before they started shipping in production devices. He agrees that the current load-tracking numbers fall short of what is really needed, but would like to see something like WALT added to the existing load-tracking code, rather than being bolted on alongside it. So the WALT patches are likely to change significantly before they can be considered for merging. 

#### Finishing out the Pixel story

Returning once again to Kjos's talk: WALT, along with SchedTune, was needed to achieve the sort of response time and power savings needed for the Pixel phone. The sched-freq CPU governor (still in use in the Pixel) needed some tweaking as well; in particular, it was changed to ramp up CPU frequencies quickly, but to lower them more slowly. That ensures that processes needing CPU time will get it right away, and the CPU won't be yanked out from underneath them too quickly afterward. 

Once the necessary software components were in place, it was time to set policies in SchedTune and the cpuset controller. The top app at any given time is set to a "spread" policy, meaning it is likely to find itself running on its own CPU. It is also given a 10% boost to ensure that it gets enough CPU time. Foreground tasks also get the "spread" policy, but without the boost, while background tasks are given a "pack" policy, causing them to be concentrated on relatively few CPUs — often just one. Kernel threads and other system processes also run with the pack policy. To further constrain placement, cpusets are used to keep background tasks on the two slower CPUs; the top app and foreground tasks share one fast CPU, while the top app gets the other fast CPU to itself. 

The results (seen toward the end of [Kjos's slides](<https://www.linuxplumbersconf.org/2016/ocw//system/presentations/3693/original/LPC-%20EAS%20for%20Android.pdf>)) show that the new scheduling code achieved or exceeded the performance of QHMP on almost all metrics. The code has been merged into the Android 3.18 and 4.4 kernel trees, and has been enabled on the Pixel device; it can also be found on the Acer R13 Chromebook and on development boards like the 96Boards HiKey. The move to the schedutil governor should happen within the next year. And, of course, the group is working to get these changes merged upstream. 

#### Epilogue: realtime scheduling

Mechanisms like SchedTune give the Android developers better control over how the scheduler responds to user-space changes. But there is another way to get low-latency response from a Linux kernel: use its realtime capabilities. Juri Lelli gave a brief talk on how Android is using realtime scheduling now, and how things may change in the near future. 

The `SCHED_FIFO` realtime scheduling class, which allows a realtime process to run uninterrupted until either it sleeps or a higher-priority realtime process comes along, is used for some [![\[Juri
Lelli\]](https://static.lwn.net/images/conf/2016/lpc/JuriLelli-sm.jpg)](<https://lwn.net/Articles/706414/>) latency-sensitive processes in Android now. In particular, the SurfaceFlinger display manager uses it, as do the audio pipeline and the threads for CPU-frequency management. There are other latency-sensitive processes that are not using realtime scheduling at this time; these include the user-interface and rendering threads. 

Why are those threads not using realtime scheduling? The scheduler's load balancing is "naive" when it comes to realtime processes, he said, so they do not get properly distributed across the CPUs. By design, the realtime scheduler will throttle processes once they use about 95% of the available CPU time; that mechanism is there to prevent a runaway realtime process from killing the system completely, but it can also get in the way when that CPU time is needed. Even with that protection, though, there is also a fear that a bug in one of those processes could cause them to take over the CPU and kill the system. 

These issues notwithstanding, the near-future hope is to move to `SCHED_FIFO` for the user-interface and rendering threads. Getting there will require a few changes, starting with tweaks to the user-space code that will hopefully be released in the Android Open Source Project in December. The realtime CPU-selection (load balancing) code will be given better energy awareness; those patches have yielded significant efficiency improvements in benchmark testing so far. There is also work on a new `TEMP_FIFO` scheduling class that would get around the throttling problem by allowing realtime tasks to continue executing, at normal priority under the completely fair scheduler, if they exceed the maximum CPU time allowed in realtime mode. 

There is an alternative to changing the behavior of `SCHED_FIFO`, though: use the [deadline scheduler](<https://lwn.net/Articles/575497/>) instead. This scheduler can allow latency-sensitive tasks to get their work done with less risk of them taking over the system. There are, however, [a number of shortcomings](<https://github.com/jlelli/sched-deadline/wiki/TODOs>) with the deadline scheduler that need to be dealt with first. These include proper control-group support and integration with the schedutil CPU-frequency governor. The deadline scheduler, too, needs a means by which a process can continue executing, at normal priority, when it exceeds the CPU time allotted to it. 

Should it be possible to fix these issues, the plan is to use the SurfaceFlinger thread as the first guinea pig for deadline scheduling. Then, perhaps, deadline scheduling will finally see a widespread, real-world deployment. 

[Thanks to LWN subscribers for supporting our travel to the event.]
