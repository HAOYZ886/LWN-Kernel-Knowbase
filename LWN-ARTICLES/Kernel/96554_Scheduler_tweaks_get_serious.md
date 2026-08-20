---
title: Scheduler tweaks get serious
url: https://lwn.net/Articles/96554/
date: "August 4, 2004"
category: Scheduler; Staircase scheduler; Voluntary preemption
author: 
---

Con Kolivas has been working on his staircase scheduler patch for a while; it was covered here [in the beginning of June](<https://lwn.net/Articles/86957/>). That scheduler found its way into the [2.6.8-rc2-mm2 patch](<https://lwn.net/Articles/96036/>), along with this comment from Andrew Morton: 

This will probably have to come out again because various people are still fiddling with the CPU scheduler. But my feeling here is that the current 1st-gen CPU scheduler has been tweaked as far as it can go and is still not 100% right. It is time to start thinking about a new design which addresses the requirements and current problems by algorithmic means rather than by tweaking. 

So it would seem that it is now open season for scheduler work. 

Initial reports on the staircase scheduler are generally - but not uniformly - good. Martin Bligh posted [some benchmark results](<https://lwn.net/Articles/96556/>) showing some significant performance improvements for the 2.6.8-rc2-mm2 kernel, especially for "low to mid loads." Ingo Molnar, instead, has [found a workload](<https://lwn.net/Articles/96562/>) which performs poorly with this scheduler; it involves running multiple processes each of which wants most, but not all, of the CPU. 

Con, meanwhile, has posted a couple of additional patches implementing additional policies in the staircase scheduler. [SCHED_BATCH](<https://lwn.net/Articles/96494/>) is another attempt at an "idle process" mode, where batch processes only run if nothing else wants the processor. This patch attempts to avoid priority inversion problems by scheduling `SCHED_BATCH` processes at normal priority when they are running in kernel mode. 

[SCHED_ISO](<https://lwn.net/Articles/96495/>), instead, is an "isochronous" mode intended for applications which need soft real-time response. Putting a process into `SCHED_ISO` is an unprivileged operation, any user can do it. Isochronous tasks start out with a relatively high priority, and should get scheduled quickly. Their allocated time slices are half of what they would otherwise be, however, and their priority drops especially quickly with CPU usage. So this mode is suitable for I/O bound processes which need to respond quickly (audio recording, CD burning, etc.), but it should not allow a hostile user to take over the system. 

Peter Williams has been working on a different set of scheduler patches. His approach is to get rid of the "expired" array (where processes go to languish when they have used up their time slices) and move everything to a single array. The patch offers two modes, being the traditional priority-based mode and a new "entitlement" mode which tries to figure how much processor time each task is entitled to, then works to ensure that each is given at least that much time. His patches are available in [a dizzying number of varieties](<https://lwn.net/Articles/96035/>); they seem to have seen less testing so far, but Andrew has said that one of them might get a turn in -mm for a while. 

Nick Piggin's [-np trees](<http://www.kerneltrap.org/~npiggin/>) also contain a new scheduler. Nick's work tries to simplify many of the scheduler calculations while retaining logic which tries to evaluate the "interactivity" of each process. Unlike some implementations, this scheduler gives longer time slices to higher-priority processes. All slices are scaled depending on the job mix, however; low-priority processes will get longer slices if there are no high-priority processes around. 

Ingo Molnar has continued his work on voluntary preemption; his [voluntary-preempt-2.6.8-rc2-O2](<https://lwn.net/Articles/95981/>) patch features a new implementation of the interrupt threads feature. The available reports indicate that, with this patch, latency problems in the 2.6 kernel are becoming few and far between. 

There is no way to tell, at this point, which of these scheduler approaches \- if any - will find its way into the mainline kernel. Evaluating schedulers takes a long time, and, for any given scheduler, there always seems to be some strange workload out there which makes it fall apart. The approaches described above (with the exception of voluntary preemption) share one nice feature, however, which is likely to argue in favor of including one of them: they all remove a significant amount of code and make the scheduler simpler and easier to understand. That, in and of itself, may be a worthwhile step toward the implementation of a top-quality Linux scheduler.
