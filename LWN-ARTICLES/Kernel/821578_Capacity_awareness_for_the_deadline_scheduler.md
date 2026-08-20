---
title: Capacity awareness for the deadline scheduler
url: https://lwn.net/Articles/821578/
date: "May 29, 2020"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "May 29, 2020 This article was contributed by Marta Rybczyńska"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

May 29, 2020

This article was contributed by Marta Rybczyńska

The Linux deadline scheduler supports realtime systems where applications need to be sure of getting their work done within a specific period of time. It allocates CPU time to deadline tasks in such a way as to ensure that each task's specific timing constraints are met. However, the current implementation does not work well on asymmetric CPU configurations like [Arm's big.LITTLE](<https://en.wikipedia.org/wiki/ARM_big.LITTLE>). Dietmar Eggemann recently [posted a patch set](<https://lwn.net/ml/linux-kernel/20200520134243.19352-1-dietmar.eggemann@arm.com/>) to address this problem by adding the notion of CPU capacity to the deadline scheduler.

In realtime systems, tasks need to meet certain timing requirements. The Linux kernel includes two realtime scheduling classes to meet the needs of these systems: POSIX realtime (often called just "realtime") and deadline.

The POSIX realtime scheduler uses task priorities as the basis of its decisions; the task with the highest priority will be run first. The deadline scheduler, instead, dispenses with priorities and describes tasks using three parameters: the run time, period, and deadline. The run time is the CPU time that the task requires to finish its immediate work, the period defines the time between two activations of the task, and the deadline is the time by which the task must be able to use its CPU time. Interested readers can find more explanation of the theory behind the Linux realtime schedulers and the differences between them in [an earlier article](<https://lwn.net/Articles/743740/>).

#### The deadline scheduler and asymmetric CPU configurations

The deadline scheduler includes an admission algorithm that runs when a task requests deadline scheduling and ensures that the system will be able to meet the task's needs. A task will be refused entry into the deadline class if the realtime guarantees cannot be met and any task in the system would miss its deadlines. However, the algorithm does not guarantee the deadlines will always be met, it only guarantees bounded tardiness for the deadline tasks and non-starvation for non-deadline ones. This is because the ability to meet deadlines in the general case depends on the tasks already in the system; a detailed explanation is available in [this article](<https://lwn.net/Articles/743946/>).

The work of the deadline scheduler becomes more complicated in asymmetric CPU configurations, like big.LITTLE or [DynamIQ](<https://www.arm.com/why-arm/technologies/dynamiq>). Such systems include different types of CPUs, with higher and lower performance. The same task running on a higher-performance ("big") CPU will take less time than when run on a lower-performance ("little") one. The deadline scheduler in current kernels does not take that difference into account, with the result that it can over-allocate the CPU time on lower-performance CPUs. Deadline tasks could end up on a little CPU, scheduled in such a way that they are unable to finish before their deadlines, while they would be able to do so on a higher-performance CPU. On such systems, the admission-control algorithm, which assumes that all CPUs perform at the level of the big ones, could overcommit the system with deadline tasks, making the system unusable. 

The information missing in the deadline scheduler is an understanding of CPU capacity — the number of instructions that can be executed in a given time. More details of how it is calculated can be found in [this article](<https://lwn.net/Articles/639543/>). The CPU capacity is already used in load balancing and in other situations, for example when changing CPU frequency [because of overheating](<https://lwn.net/Articles/788380/>), and it has recently been [added to the realtime scheduler](<https://git.kernel.org/linus/804d402fb6f6>). Eggemann's work takes capacity into account in the deadline scheduler's admission-control and task-placement algorithms. After the changes, the deadline scheduler places the tasks in such a way that the available capacity is sufficient to allow tasks to meet their deadlines in both symmetric and asymmetric CPU configurations.

#### The changes

The admission-control algorithm bases its decisions on the total CPU capacity provided by the system. In symmetric systems, where all CPUs have the same capacity, the sum is the simply the number of CPUs multiplied by a constant. Current kernels calculate total capacity this way even on asymmetric systems; all CPUs are assumed to have the capacity of the largest ones. The new code changes this metric in the asymmetric case, causing it to calculate the sum of the actual capacities of all active CPUs.

The deadline scheduler's task-placement code also must gain a better understanding of the system's CPU topology. Before moving a task to a new CPU, the scheduler needs to ensure that the new CPU can handle that task. In asymmetric systems, a new type of a check is needed to find out if the CPU's capacity is sufficient to perform a given task's work before its deadline. This fitness check is performed using the following formula:

```
(CPU capacity) / 1024 >= (task runtime) / (task deadline)
```

The default CPU capacity is 1024; lower-performance CPUs have a capacity lower than that. The left-hand side of this formula thus yields a fraction indicating the relative capacity of the CPU in question. For example, assume the capacity of a small CPU is 462, then that fraction is 462/1024, or 0.45. The formula will admit only tasks that have ratio of run time (which is relative to a big CPU) to deadline of less or equal to 0.45. A task with a runtime of 13,000µs and deadline of 16,000µs will not be admitted, as 13,000/16,000 is 0.81, which is a larger value.

This check is used when waking up a deadline task, moving a deadline task to a different CPU, and migrating a task out of a CPU that is going offline. Eggemann [showed some example capacity calculations](<https://lwn.net/ml/linux-kernel/b72f72c1-6043-c44f-31f5-fb4137bd5d52@arm.com/>) during the discussion of an earlier version of the patch set.

If a task cannot be served according to its deadline, it will miss that deadline. This can always happen with the deadline scheduler, for example, in a situation when the task was admitted successfully, but then one of the CPUs went offline and the overall capacity of the system was reduced. [This change](<https://lwn.net/ml/linux-kernel/20200427083709.30262-7-dietmar.eggemann@arm.com/>), introduced in the patch set, is how this situation will be handled in an asymmetric configuration; the scheduler will choose the CPU with the maximum available capacity. If there are several of them, it prefers the current CPU, if possible, to make use of any remaining cache contents. 

#### Limitations and further work

The work on asymmetric CPU support in the deadline scheduler does not end here. The current patch set only supports the case when there is at least one CPU without any deadline tasks; otherwise task placement may still be incorrect. The case of more heavily loaded systems will need to be addressed later.

During the discussion, Juri Lelli [pointed out](<https://lwn.net/ml/linux-kernel/20200427133438.GA6469@localhost.localdomain/>) a possible problem: if a set of small deadline tasks starts first, they will be placed on large CPUs. If a bigger task is then admitted, it may not find a CPU large enough to run on. Luca Abeni (co-author of the patch set) [responded](<https://lwn.net/ml/linux-kernel/20200427161715.3dd3a148@nowhere/>) that they do have an updated patch where the scheduler places tasks on the smallest CPU that can get the work done. This patch will be submitted later.

The patch set has received positive reviews, and we should expect that this fix will become part of the mainline kernel soon. We can also expect to see more patches in this area as there is more work to do; with more asymmetric CPU architectures popping up, users may require better support of such configurations in their workloads.
