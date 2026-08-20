---
title: "Removing the scheduler's energy-margin heuristic"
url: https://lwn.net/Articles/899303/
date: "July 1, 2022"
category: "Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet July 1, 2022"
---

By **Jonathan Corbet**  
July 1, 2022

The CPU scheduler's job has never been easy; it must find a way to allocate CPU time to all tasks in the system that is fair, allows all tasks to progress, and maximizes the throughput of the system as a whole. More recently, it has been called upon to satisfy another constraint: minimizing the system's energy consumption. There is currently [a patch set](<https://lwn.net/ml/linux-kernel/20220621090414.433602-1-vdonnefort@google.com/>) in circulation, posted by Vincent Donnefort with work from Dietmar Eggemann as well, that changes how this constraint is met. The actual change is small, but it illustrates how hard it can be to get the needed heuristics right. 

Reduction of energy use is, of course, a worthy goal; energy that is not wasted becomes available for the mining of more cryptocurrency, after all. There are some smaller considerations as well, such as environmental benefits, that justify the effort, but the proliferation of battery-powered devices has added more urgency to the task. If batteries can be made to last longer, doomscrolling interruptions will be fewer and users will be happier. 

These pressures have led to the addition of energy-aware scheduling to the kernel. When the scheduler considers the placement of tasks in the system, it will work to reduce the amount of energy consumed overall; this work includes running the CPUs at the power level that is the most efficient for the current load and powering down processors entirely when possible. For example, if a CPU that is currently running at a given power level can accept another task without having to move to a higher power level, it may make sense to move a task there from another CPU. 

In 2018, [this patch from Quentin Perret](<https://git.kernel.org/linus/732cd75b8c92>) (which was part of the [energy-aware scheduling patch set](<https://lwn.net/ml/linux-kernel/20181203095628.11858-1-quentin.perret@arm.com/>)) added a function called `find_energy_efficient_cpu()` to the scheduler; its job was to find the best place (from an energy-consumption point of view) for a given task. The heuristic used, at its core, is to find the least-busy CPU within each "performance domain" (cluster of CPUs whose energy usage is tied together) and estimate the energy cost (or savings) that would result from putting the task on that CPU. The least-busy CPU is the most likely to stay in a low-power state, so it makes a logical target for some extra work. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

There is, however, a cost to moving a task from one CPU to another; that task may leave some or all of its memory caches behind, which will slow it down. That affects performance and is not good for energy use either, so it should be avoided whenever possible. As a way of preventing excess task movement between CPUs, `find_energy_efficient_cpu()` would only move a task if the result would be a savings of at least 6% of the energy used by the task's previous CPU. 

The calculation of the best CPU was expensive, though, to the point where it was adding unwanted latency to scheduling decisions. So Perret [reworked it](<https://git.kernel.org/linus/eb92692b2544>) for the 5.4 kernel release in 2019. The intent was to get the same results for less CPU cost; the patch changelog said ""no functional changes intended"". It turns out that there was a subtle change, though, that apparently escaped review: the 6% rule now compared against the energy used by the entire system, rather than just the previous CPU a task was running on. That is a relatively high bar to movement that, on a system with enough CPUs, could become impossible for a task to meet. 

Even on smaller systems, the new rule effectively prevents task movement in many cases. This is especially true in situations where there are a relatively large number of small tasks running — a situation that is often found on Android devices, where energy efficiency is a real concern. If it is no longer able to move tasks to save energy, all of the work done by `find_energy_efficient_cpu()` is wasted and the device runs less efficiently than it otherwise would. 

An obvious solution would be to undo the 5.4 change to the algorithm but, Donnefort [said](<https://lwn.net/ml/linux-kernel/20220621090414.433602-8-vdonnefort@google.com/>), ""the original version didn't have strong grounds either"". Indeed, there was never any reasoning given for the 6% number, which was 1.5% until it was raised to 6% in [version 4 of the patch set](<https://lwn.net/ml/linux-kernel/20180628114043.24724-1-quentin.perret%40arm.com/>). Its best feature may be that it is relatively easy to approximate with a right-shift operation. The conclusion Donnefort reached is that it would be better to simply remove that test entirely and migrate a task whenever it appears that the move would result in reduced energy consumption. 

According to benchmarks posted with the patch set, the result is indeed better energy performance — up to a 5.6% reduction on a video benchmark. CPU performance is reduced slightly in some tests, but the change does not seem to be significant. As Donnefort put it: 

> The margin removal lets the kernel make the best use of the Energy Model, tasks are more likely to be placed where they fit and this saves a substantial amount of energy, while having a limited impact on performance. 

One possible drawback of this change could be increased bouncing of tasks between CPUs, but Donnefort said that testing ""showed no issue"". 

This patch set is in its eleventh revision, having seen a number of changes in response to review comments. In response to this version, scheduler maintainer Peter Zijlstra had [just one word](<https://lwn.net/ml/linux-kernel/Yrqr84h5xe2GYMVb@hirez.programming.kicks-ass.net/>) to say: ""Thanks!"". So it would appear that removal of the 6% heuristic makes sense, and that it will be finding its way into the mainline sooner rather than later.
