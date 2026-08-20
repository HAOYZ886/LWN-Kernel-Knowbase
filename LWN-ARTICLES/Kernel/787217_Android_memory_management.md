---
title: Android memory management
url: https://lwn.net/Articles/787217/
date: "May 1, 2019"
category: "Android; Memory management-Out-of-memory handling"
author: "By Jonathan Corbet May 1, 2019 LSFMM"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
May 1, 2019

* * *

[LSFMM](<https://lwn.net/Articles/lsfmm2019/>)

The Android system is designed to provide a responsive user experience on systems that, in a relative sense at least, have limited amounts of CPU and memory. Doing so requires a number of techniques, including regular use of a low-memory process killer, that are not seen elsewhere. In a memory-management-track session at the 2019 Linux Storage, Filesystem, and Memory-Management Summit, Suren Baghdasaryan covered a number of issues related to how Android ensures that interactive processes have enough memory to get their jobs done. 

Baghdasaryan started by noting that the recently added [pressure-stall information](<https://lwn.net/Articles/759781/>) feature, which was not originally developed for Android at all, has proved to be quite useful. It gives the Android runtime more accurate information about memory pressure, which can be used to better manage the set of running processes. Overall, he said, the goal of Android memory management is to ensure that interactive processes work as well as possible while minimizing the number of out-of-memory kills needed to do that. 

The Android [low-memory killer daemon](<https://source.android.com/devices/tech/perf/lmkd>) (LMKD) is charged with making all of this happen. Beyond pressure-stall information, a number of recent developments are helping to make it more effective at that task. File descriptors that represent [![\[Suren
Baghdasaryan\]](https://static.lwn.net/images/conf/2019/lsfmm/SurenBaghdasaryan-sm.jpg)](<https://lwn.net/Articles/787221/>) processes ([discussed here](<https://lwn.net/Articles/784831/>)) are helpful, and the upcoming ability to [poll those descriptors](<https://lwn.net/ml/linux-kernel/20190425190010.46489-1-joel@joelfernandes.org/>) for process death will also be useful. But there are still issues with reclaiming memory when the need arises. The use of control groups, while helpful in many ways, does split the kernel's least-recently-used (LRU) list into a large number of smaller lists, which makes reclaim harder in general. 

The core issue discussed in this session, though, was quick reclaim of memory from processes that have been killed by LMKD. Depending on what else is happening in the system, that reclaim can take a long and unpredictable time, which makes LMKD's problem harder and forces it to kill processes sooner than it otherwise would. Baghdasaryan's [opportunistic reclaim patches](<https://lwn.net/Articles/785709/>) are an attempt to improve this situation; this feature tries to immediately strip memory from a process that has been explicitly killed. That avoids situations where the target process itself may be slow to free those resources, making reclaim faster and more predictable. 

The first implementation was based on the [OOM reaper](<https://lwn.net/Articles/668126/#reaper>) code which, he said, is probably not how a final implementation should look. But getting to that final version requires answering a few questions, the first of which was where the work of reaping of memory from a killed process should be done. One option is to have the process sending the `SIGKILL` signal take responsibility (in kernel space) for this reclaim work. There are a number of advantages to doing things that way: it is relatively simple to implement, the CPU time required will be charged to the killing process, priority inheritance for the reclaim work will happen automatically, and it provides for better user-space control over when this work is done. On the other hand, it may require a new user-space API to control opportunistic reclaim, and the scalability of reclaim from large processes could be a problem. 

The alternative would be to do this work in one or more kernel threads. That simplifies the API issues and might make things more scalable. That, however, takes away any sort of user control over when expedited reclaim might happen. 

Rik van Riel observed that this is, at its core, a hardware problem. At some point, mobile devices will get fast enough, with enough memory, that the reclaim problem will go away by itself. In that case, adding a new API to speed up reclaim might well be the wrong thing to do. Michal Hocko, though, said that the real problem is processes that are blocked (and thus cannot do their own cleanup) rather than hardware limitations. 

Johannes Weiner said that, to the extent that hardware is the problem, things could be improved by automatically moving exiting processes to the fastest CPU in the system. Other resource limits are waived on exiting processes so they can get out of the way quickly, he said, so it might make sense to do the same with regard to CPU placement. Others worried that this could create power-consumption issues, but since this situation tends to come about when an interactive process wants to run, a fast CPU should be running and available anyway. 

Hocko replied that power use is not a big concern, but that process isolation might be. If a process has been confined to a slow CPU, moving it to a fast one, even if it's just to die, may break the isolation between groups and affect the running of interactive processes. If the desire is to have a killed process do its cleanup on a fast CPU, the solution is for user space to explicitly move the process to a different control group prior to killing it. 

Mel Gorman said that there are a couple of problems that need to be addressed here. One is that there is not enough CPU time to do the cleanup work quickly. But the other aspect is that address spaces have gotten so large that even the fastest CPU is limited in how quickly it can get the job done. But the best solution is simple, he said, at least on the kernel side: an exiting process should be migrated to a fast CPU if its CPU mask allows that. The rest of the problem can be solved in user space, which can move the process prior to killing it if reclaim time is an issue. Doing anything else in the kernel would, he said, break isolation for somebody eventually. 

Matthew Wilcox returned to the issue of doing the reclaim in the context of the killing process, which skirts around a number of these issues. Gorman replied that implementing reclaim that way is guaranteed to increase the number of inter-processor interrupts, since the killed process's memory will have been touched on two separate CPUs. That would not be good for performance There are also concerns that doing reclaim this way would turn the `kill()` system call into a blocking operation that could run for an arbitrary time, which could surprise callers. 

One final option that was discussed was a remote version of the `madvise(MADV_DONTNEED)` operation; that would allow one process to force reclaim of (some of) another process's memory. Gorman worried, though, that this operation would have a large "potential for shenanigans"; this concern could be addressed by applying the usual "can the calling process use `ptrace()` on the target?" test. This call would have the advantage that it could be done in multiple threads, each of which would release a part of the address space; that would be a simple way of parallelizing the task. At the close of the session, it was also suggested that either `fadvise()` or `truncate()`, when called on a process file descriptor, could be given the effect of reclaiming that process's memory, but the idea was not developed further.
