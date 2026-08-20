---
title: Deadline scheduling part 1 — overview and theory
url: https://lwn.net/Articles/743740/
date: "January 16, 2018"
category: "Realtime-Deadline scheduling; Scheduler-Deadline scheduling"
author: "January 16, 2018 This article was contributed by Daniel Bristot de Oliveira"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

January 16, 2018

This article was contributed by Daniel Bristot de Oliveira

Realtime systems are computing systems that must react within precise time constraints to events. In such systems, the correct behavior does not depend only on the logical behavior, but also in the timing behavior. In other words, the response for a request is only correct if the logical result is correct and produced within a deadline. If the system fails to provide the response within the deadline, the system is showing a defect. In a multitasking operating system, such as Linux, a realtime scheduler is responsible for coordinating the access to the CPU, to ensure that all realtime tasks in the system accomplish their job within the deadline. 

The deadline scheduler enables the user to specify the tasks' requirements using well-defined realtime abstractions, allowing the system to make the best scheduling decisions, guaranteeing the scheduling of realtime tasks even in higher-load systems. 

This article provides an introduction to realtime scheduling and some of the theory behind it. The second installment will be dedicated to the Linux deadline scheduler in particular. 

#### Realtime schedulers in Linux

Realtime tasks differ from non-realtime tasks by the constraint of having to produce a response for an event within a deadline. To schedule a realtime task to accomplish its timing requirements, Linux provides two realtime schedulers: the POSIX realtime scheduler (henceforth called the "realtime scheduler") and the deadline scheduler. 

The POSIX realtime scheduler, which provides the FIFO (first-in-first-out) and RR (round-robin) scheduling policies, schedules each task according to its fixed priority. The task with the highest priority will be served first. In realtime theory, this scheduler is classified as a fixed-priority scheduler. The difference between the FIFO and RR schedulers can be seen when two tasks share the same priority. In the FIFO scheduler, the task that arrived first will receive the processor, running until it goes to sleep. In the RR scheduler, the tasks with the same priority will share the processor in a round-robin fashion. Once an RR task starts to run, it will run for a maximum quantum of time. If the task does not block before the end of that time slice, the scheduler will put the task at the end of the round-robin queue of the tasks with the same priority and select the next task to run. 

In contrast, the deadline scheduler, as its name says, schedules each task according to the task's deadline. The task with the earliest deadline will be served first. Each scheduler requires a different setup for realtime tasks. In the realtime scheduler, the user needs to provide the scheduling policy and the fixed priority. For example: 

```
chrt -f 10 video_processing_tool
```

With this command, the `video_processing_tool` task will be scheduled by the realtime scheduler, with a priority of 10, under the FIFO policy (as requested by the `-f` flag). 

In the deadline scheduler, instead, the user has three parameters to set: the period, the run time, and the deadline. The period is the activation pattern of the realtime task. In a practical example, if a video-processing task must process 60 frames per second, a new frame will arrive every 16 milliseconds, so the period is 16 milliseconds. 

The run time is the amount of CPU time that the application needs to produce the output. In the most conservative case, the runtime must be the worst-case execution time (WCET), which is the maximum amount of time the task needs to process one period's worth of work. For example, a video processing tool may take, in the worst case, five milliseconds to process the image. Hence its run time is five milliseconds. 

The deadline is the maximum time in which the result must be delivered by the task, relative to the period. For example, if the task needs to deliver the processed frame within ten milliseconds, the deadline will be ten milliseconds. 

It is possible to set deadline scheduling parameters using the `chrt` command. For example, the above-mentioned tool could be started with the following command: 

```
chrt -d --sched-runtime 5000000 --sched-deadline 10000000 \
        	    --sched-period 16666666 0 video_processing_tool
```

Where: 

  * `--sched-runtime 5000000` is the run time specified in nanoseconds 
  * `--sched-deadline 10000000` is the relative deadline specified in nanoseconds. 
  * `--sched-period 16666666` is the period specified in nanoseconds 
  * `0` is a placeholder for the (unused) priority, required by the `chrt` command 

In this way, the task will have a guarantee of 5ms of CPU time every 16.6ms, and all of that CPU time will be available for the task before the 10ms deadline passes. 

Although the deadline scheduler's configuration looks complex, it is not. By giving the correct parameters, which are only dependent on the application itself, the user does not need to be aware of all the other tasks in the system to be sure that the application will deliver its results before the deadline. When using the realtime scheduler, instead, the user must take into account all of the system's tasks to be able to define which is the correct fixed priority for any task. 

Since the deadline scheduler knows how much CPU each deadline task will need, it knows when the system can (or cannot) admit new tasks. So, rather than allowing the user to overload the system, the deadline scheduler denies the addition of more deadline tasks, guaranteeing that all deadline tasks will have CPU time to accomplish their tasks with, at least, a bounded tardiness. 

In order to further discuss benefits of the deadline scheduler it is necessary to take a step back and look at the bigger picture. To that end, the next section explains a little bit about realtime scheduling theory. 

#### A realtime scheduling overview

In scheduling theory, realtime schedulers are evaluated by their ability to schedule a set of tasks while meeting the timing requirements of all realtime tasks. In order to provide deterministic response times, realtime tasks must have a deterministic timing behavior. The task model describes the deterministic behavior of a task. 

Each realtime task is composed of _N_ recurrent activations; a task activation is known as a **job**. A task is said to be **periodic** when a job takes place after a fixed offset of time from its previous activation. For instance, a periodic task with period of 2ms will be activated every 2ms. Tasks can also be **sporadic**. A sporadic task is activated after, at least, a minimum inter-arrival time from its previous activation. For instance, a sporadic task with a 2ms period will be activated after at least 2ms from the previous activation. Finally, a task can be **aperiodic** , when there is no activation pattern that can be established. 

Tasks can have an **implicit deadline** , when the deadline is equal to the activation period, or a **constrained deadline** , when the deadline can be less than (or equal to) the period. Finally, a task can have an **arbitrary deadline** , where the deadline is unrelated to the period. 

Using these patterns, realtime researchers have developed ways to compare scheduling algorithms by their ability to schedule a given task set. It turns out that, for uniprocessor systems, the Early Deadline First (EDF) scheduler was found to be optimal. A scheduling algorithm is optimal when it fails to schedule a task set only when no other scheduler can schedule it. The deadline scheduler is optimal for periodic and sporadic tasks with deadlines less than or equal to their periods on uniprocessor systems. Actually, for either periodic or sporadic tasks with implicit deadlines, the EDF scheduler can schedule any task set as long as the task set does not use more than 100% of the CPU time. The Linux deadline scheduler implements the EDF algorithm. 

Consider, for instance, a system with three periodic tasks with deadlines equal to their periods: 

> Task | Runtime  
> (WCET) | Period  
> ---|---|---  
> T1 | 1 | 4  
> T2 | 2 | 6  
> T3 | 3 | 8  
  
The CPU time utilization (U) of this task set is less than 100%: 

```
U =  1/4 + 2/6 + 3/8 = 23/24
```

For such a task set, the EDF scheduler would present the following behavior: 

> ![\[Scheduling chart\]](https://static.lwn.net/images/2018/deadline/dl.png)

However, it is not possible to use a fixed-priority scheduler to schedule this task set while meeting every deadline; regardless of the assignment of priorities, one task will not run in time to get its work done. The resulting behavior will look like this: 

> ![\[Scheduling chart\]](https://static.lwn.net/images/2018/deadline/fp.png)

The main advantage of deadline scheduling is that, once you know each task's parameters, you do not need to analyze all of the other tasks to know that your tasks will all meet their deadlines. Deadline scheduling often results in fewer context switches and, on uniprocessor systems, deadline scheduling is able to schedule more tasks than fixed priority-scheduling while meeting every task's deadline. However, the deadline scheduler also has some disadvantages. 

The deadline scheduler provides a guarantee of accomplishing each task's deadline, but it is not possible to ensure a minimum response time for any given task. In the fixed-priority scheduler, the highest-priority task always has the minimum response time, but that is not possible to guarantee with the deadline scheduler. The EDF scheduling algorithm is also more complex than fixed-priority, which can be implemented with O(1) complexity. In contrast, the deadline scheduler is O(log(n)). However, the fixed-priority requires an “offline computation” of the best set of priorities by the user, which can be as complex as O(N!). 

If, for some reason, the system becomes overloaded, for instance due to the addition of a new task or a wrong WCET estimation, it is possible to face a domino effect: once one task misses its deadline by running for more than its declared run time, all other tasks may miss their deadlines as shown by the regions in red below: 

> ![\[Domino effect\]](https://static.lwn.net/images/2018/deadline/domino.png)

In contrast, with fixed-priority scheduling, only the tasks with lower priority than the task which missed the deadline will be affected. 

In addition to the prioritization problem, multi-core systems add an allocation problem. On a multi-core system, the scheduler also needs to decide where the tasks can run. Generally, the scheduler can be classified as one of the following: 

> ![\[Scheduler types\]](https://static.lwn.net/images/2018/deadline/schedtypes.png)

  * **Global** : When a single scheduler manages all M CPUs of the system. In other words, tasks can migrate to all CPUs. 
  * **Clustered** : When a single scheduler manages a disjoint subset of the M CPUs. In other words, tasks can migrate to just a subset of the available CPUs. 
  * **Partitioned** : When each scheduler manages a single CPU, so no migration is allowed. 
  * **Arbitrary** : Each task can run on an arbitrary set of CPUs. 

In multi-core systems, global, clustered, and arbitrary deadline schedulers are not optimal. The theory for multi-core scheduling is more complex than for single-core systems due to many anomalies. For example, in a system with M processors, it is possible to schedule M tasks with a run time equal to the period. For instance, a system with four processors can schedule four "BIG" tasks with both run time and period equal to 1000ms. In this case, the system will reach the maximum utilization of: 

```
4 * 1000/1000 = 4
```

The resulting scheduling behavior will look like: 

> ![\[Four big tasks\]](https://static.lwn.net/images/2018/deadline/bigs.png)

It is intuitive to think that a system with a lower load will be schedulable too, as it is for single-processor systems. For example, in a system with four processors, a task set composed of four small tasks with the minimum runtime, let's say 1ms, at every 999 milliseconds period, and just one task BIG task, with runtime and period of one second. The load of this system is: 

```
4 * (1/999) + 1000/1000 = 1.004
```

As 1.004 is smaller than four, intuitively, one might say that the system is schedulable, But that is not true for global EDF scheduling. That is because, if all tasks are released at the same time, the M small tasks will be scheduled in the M available processors. Then, the big task will be able to start only after the small tasks have run, hence finishing its computation after its deadline. As illustrated below. This is known as the Dhall's effect. 

> ![\[Dhall's effect\]](https://static.lwn.net/images/2018/deadline/dhall.png)

Distribution of tasks to processors turns out to be an NP-hard problem (a bin-packing problem, essentially) and, due to other anomalies, there is no dominance of one scheduling algorithm over any others. 

With this background in place, we can turn to the details of the Linux deadline scheduler and the best ways to take advantage of its capabilities while avoiding the potential problems. See [the second half of this series](<https://lwn.net/Articles/743946/>), to be published soon, for the full story.
