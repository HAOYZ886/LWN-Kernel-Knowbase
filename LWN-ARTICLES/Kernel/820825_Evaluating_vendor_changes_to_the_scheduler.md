---
title: Evaluating vendor changes to the scheduler
url: https://lwn.net/Articles/820825/
date: "May 19, 2020"
category: "Android; Power management-CPU scheduling; Scheduler-and power management"
author: "By Jonathan Corbet May 19, 2020 OSPM"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
May 19, 2020

* * *

[OSPM](<https://lwn.net/Articles/820337/>)

The kernel's CPU scheduler does its best to make the right decisions for just about any workload; over the years, it has been extended to better handle mobile-device scheduling as well. But handset vendors still end up applying their own patches to the scheduler for the kernels they ship. Shipping out-of-tree code in this way leads to a certain amount of criticism from the kernel community but, as Vincent Donnefort pointed out in his session at the 2020 [Power Management and Scheduling in the Linux Kernel summit](<http://retis.sssup.it/ospm-summit/>) (OSPM), those patches are applied for a reason. He looked at a set of vendor scheduler patches to see why they are being used. 

As a testbed for these patches, Donnefort chose the Pixel 4 phone. It's a device with good upstream support, so it's easy to replace its kernel [![\[Vincent
Donnefort\]](https://static.lwn.net/images/conf/2020/ospm/VincentDonnefort-sm.png)](<https://lwn.net/Articles/820827/>) without the need for lots of other out-of-tree code. This device has three different CPU core sizes, small, medium, and large, where the small cores are small indeed. It is imperative to pick the correct CPU for any given task, or there will be a cost to pay in performance or energy use. The [PCMark](<https://en.wikipedia.org/wiki/PCMark>) benchmark was used to evaluate performance, while power measurement was done directly from the phone's power rails. A 4.14 kernel was used for the tests. 

The first patch tested performs CPU isolation by actively evacuating tasks to other CPUs; the intent is to idle the CPU and let it be put into a sleep state. Tasks are migrated, interrupts are directed elsewhere, and the CPU is removed from the load balancer's attention entirely; kernel threads attached to that CPU still run, though. This is, he said, a sort of lightweight form of CPU hotplug. 

This patch works by looking at the load presented by all of the running tasks and calculating how much CPU power is needed. If the number of running CPUs exceeds what is needed, it will try to isolate one or more of them. This decision is made in user space. 

In performance testing, Donnefort found that CPU isolation reduces throughput slightly, but also gives a 4% drop in power consumption. Vincent Guittot asked why the energy model built into the kernel couldn't handle this task; Donnefort responded that he didn't try to evaluate alternative solutions to the problem. The results show, though, that there is room for improvement. 

The other patches were presented as a set. They were: 

  * "Migration margins": this patch changes the way the kernel picks a CPU for a task on an asymmetric system. This is done by comparing the task's expected utilization with the capacity of the CPU; the mainline kernel will only place a task on a CPU if there will be 20% of its capacity left afterward. The vendor patch lowers this margin to 5%, thus increasing the chance that a given task will end up on a smaller, more energy-efficient CPU. 
  * There is a change to how the scheduler does task packing. The mainline tries to keep tasks contained within a single cluster (thus possibly allowing other clusters to go idle), but will try to spread out tasks across the CPUs in a cluster. The vendor patch, instead, works harder to pack tasks into a single CPU, though stopping before it would become necessary to increase the CPU's frequency. 
  * The mainline puts some effort into finding the most efficient CPU to run any given task on — too much time, it seems, for some vendors, who make a change to that algorithm. With this change, the kernel decides where to put a task by first looking at where it was running last time; if that CPU is idle and the task fits there, the placement logic will be shorted out and that CPU will be chosen immediately. He noted that energy-aware task placement has improved considerably since the release of the 4.14 kernel used for these tests. 
  * When placing a realtime task, the kernel performs a search for the CPU that is running the lowest-priority task; that will be the easiest one to preempt. The vendor patch expands this search to look at utilization and idle states as well, trying to find the CPU that is the least busy overall. The search is also biased toward finding the smallest suitable CPU. 

The benchmark results for each of these patches were remarkably similar. They all tended to hurt performance by 3-5% while reducing energy use by 8-11%. What Donnefort did not do, though, was to benchmark a system with all of them applied; he cautioned against assuming that those differences would be additive with all of the patches in the system. 

He concluded with the simple assertion that, even if some of these changes are controversial, they are clearly useful in this setting. He will be looking at ways of getting those changes into an acceptable form for merging upstream. 

In the discussion, Qais Youssef said that some of his recent CPU-capacity changes might be able to replace some of this work. Dietmar Eggemann asked why the energy model wasn't providing CPU isolation now; it should already be pushing things aggressively toward small CPUs. Peter Zijlstra agreed that it was important to figure out why that workaround was necessary; perhaps the scheduler should look more closely at idle states in the energy-aware path. Donnefort said that CPU isolation in this form is probably not the right solution for the mainline kernel, but it does show that there is something to be gained that way. 

See [Donnefort's slides [PDF]](</images/conf/2020/ospm/donnefort-slides.pdf>) for detailed results and more.
