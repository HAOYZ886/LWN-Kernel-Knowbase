---
title: "LC-Asia: A big LITTLE MP update"
url: https://lwn.net/Articles/541005/
date: "March 6, 2013"
category: "Architectures-Arm; big.LITTLE"
author: "By Jonathan Corbet March 6, 2013"
---

By **Jonathan Corbet**  
March 6, 2013

The ARM "big.LITTLE" architecture pairs two types of CPU — fast, power-hungry processors and slow, efficient processors — into a single package. The result is a system that can efficiently run a wide variety of workloads, but there is one little problem: the Linux kernel currently lacks a scheduler that is able to properly spread a workload across multiple types of processors. Two approaches to a solution to that problem are being pursued; a session at the 2013 Linaro Connect Asia event reviewed the current status of the more ambitious of the two. 

LWN recently looked at the [big.LITTLE switcher](<https://lwn.net/Articles/539840/>), which pairs fast and slow processors and uses the CPU frequency subsystem to switch between them. The switcher approach has the advantage of being relatively straightforward to get working, but it also has a disadvantage: only half of the CPUs in the system can be doing useful work at any given time. It also is not yet posted for review or merging into the mainline, though this posting is said to be planned for the near future, after products using this code begin to ship. 

The alternative approach has gone by the name "big LITTLE MP". Rather than play CPU frequency governor games, big LITTLE MP aims to solve the problem directly by teaching the scheduler about the differences between processor types and how to distribute tasks between them. The big.LITTLE switcher patch touches almost no files outside of the ARM architecture subtree; the big LITTLE MP patch set, instead, is focused almost entirely on the core scheduler code. At Linaro Connect Asia, developers Vincent Guittot and Morten Rasmussen described the current state of the patch set and the plans for getting it merged in the (hopefully) not-too-distant future. 

The big LITTLE MP patch set has recently seen a major refactoring effort. The first version was strongly focused on the heterogeneous multiprocessing (HMP) problem but, among other things, it is hard to get developers for the rest of the kernel interested in HMP. So the new patch set aims to improve [![\[Morten and
Vincent\]](https://static.lwn.net/images/conf/2013/lc-asia/guittot-rasmussen-sm.jpg)](<https://lwn.net/Articles/541154/>) scheduling results on all systems, even traditional SMP systems where all CPUs are the same. There is a patch set that is in internal review and available on the [Linaro git server](<http://git.linaro.org/gitweb?p=arm/big.LITTLE/mp.git;a=summary>). Some parts have been publicly posted recently; soon the rest should be more widely circulated as well. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The new patches are working well; for almost all workloads, their performance is similar to that achieved with the old patch set. The patches were developed with a view toward simplicity: they affect a critical kernel path, so they must be both simple and fast. Some of the patches, fixes for the existing scheduler, have already been posted to the mailing lists. The rest try to augment the kernel's scheduler with three simple rules: 

  * Small tasks (those that only use small amounts of CPU time for brief periods) are not worth the trouble to schedule in any sophisticated way. Instead, they should just be packed onto a single, slow core whenever they wake up, and kept there if at all possible. 

  * Load balancing should be concerned with the disposition of long-running tasks only; it should simply pass over the small tasks. 

  * Long-running tasks are best placed on the faster cores. 

Implementing these policies requires a set of a half-dozen patches. One of them is the "small-task packing" patch that was [covered here](<https://lwn.net/Articles/520857/>) in October, 2012. Another works to expand the use of [per-entity load tracking](<https://lwn.net/Articles/531853/>) (which is currently only enabled when control groups and the CPU controller are being used) so that the per-task load values are always available to the scheduler. A further patch ensures that the "LB_MIN" scheduler feature is turned on; LB_MIN (which defaults to "off" in mainline kernels) causes the load balancer to pass over small tasks when working to redistribute the computing load on the system, essentially implementing the second policy objective above. 

After that, the patch set augments the scheduler with the concept of the "capacity" of each CPU; the unloaded capacity is essentially the clock speed of the processor. The load balancer is tweaked to migrate processes to the CPU with the largest available capacity. This task is complicated by the fact that a CPU's capacity may not be a constant value; realtime scheduling, in particular, can "steal" capacity away from a CPU to give to realtime-priority tasks. Scheduler domains also need to be tuned for the big.LITTLE environment with an eye toward reducing the periodic load balancing work that needs to be done. 

The final piece is not yet complete; it is called "scheduling invariance." Currently, the "load" put on the system by a process is a function of the amount of time that process spends running on the CPU. But if some CPUs are faster than others, the same process could end up with radically different load values depending on which CPU it is actually running on. That is suboptimal; the actual amount of work the process needs to do is the same in either case, and varying load values can cause the scheduler to make poor decisions. For now, the problem is likely to be solved by scaling the scheduler's load calculations by a constant value associated with each processor. Processes running on a CPU that is ten times faster than another will accumulate load ten times more quickly. 

Even then, the load calculations are not perfect for the HMP scheduling problem because they are scaled by the process's priority. A high-priority task that runs briefly can look like it is generating as much load as a low-priority task that runs for long periods, but the scheduler may want to place those processes in different ways. The best solution to this problem is not yet clear. 

A question from the audience had to do with testing: how were the developers testing their scheduling decisions? In particular, was the [Linsched](<https://lwn.net/Articles/487701/>) testing framework being used? The answer is that no, Linsched is not being used. It has not seen much development work since it was posted for the 3.3 kernel, so it does not work with current kernels. Perhaps more importantly, its task representation is relatively simple; it is hard to present it with something resembling a real-world Android workload. It is easier, in the end, to simply monitor a real kernel with an actual Android workload and see how well it performs. 

The plan seems to be to post a new set of big LITTLE MP patches in the near future with an eye toward getting them upstream. The developers are a little concerned about that; getting reviewer attention for these patches has proved to be difficult thus far. Perhaps persistence and a more general focus will help them to get over that obstruction, clearing the way for proper scheduling on heterogeneous multiprocessor systems in the not-too-distant future. 

[Your editor would like to thank Linaro for travel assistance to attend this event.]
