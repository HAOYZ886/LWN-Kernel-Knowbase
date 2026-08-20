---
title: "Reconsidering the scheduler's wake_wide() heuristic"
url: https://lwn.net/Articles/728942/
date: "July 27, 2017"
category: Scheduler
author: "July 27, 2017 This article was contributed by Matt Fleming"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

July 27, 2017

This article was contributed by [Matt Fleming](<http://www.codeblueprint.co.uk/>)

The kernel's CPU scheduler is charged with choosing which task to run next, but also with deciding where in a multi-CPU system that task should run. As is often the case, that choice comes down to heuristics — rules of thumb codifying the developers' experience of what tends to work best. One key task-placement heuristic has been in place since 2015, but a recent discussion suggests that it may need to be revisited. 

Scheduler wakeups happen all the time. Tasks will often wait for an event (e.g. timer expiration, POSIX signal, `futex()` system call, etc.); a wakeup is sent when the event occurs and the waiting task resumes execution. The scheduler's job is to find the best CPU to run the task being woken. Making the correct choice is crucial for performance. Some message-passing workloads benefit from running tasks on the same CPU, for example; the [pipetest](<http://people.redhat.com/mingo/cfs-scheduler/tools/pipe-test.c>) micro-benchmark is a simple model of that kind of workload. Pipetest uses two communicating tasks that take turns sending and receiving messages; the tasks never need to run in parallel and thus perform best if their data is in the cache of a single CPU.

In practice, many workloads do not communicate in lockstep — in fact most workloads do not take turns sending messages. In highly parallel applications, messages are sent at random times in response to external input. A typical communication scheme is the producer-consumer model, where one master task wakes up multiple slave tasks. These workloads perform better if the tasks run simultaneously on different CPUs. But modern machines have lots of CPUs to choose from when waking tasks. The trouble is picking the best one.

The choice of CPU also affects power consumption. Packing tasks onto fewer CPUs allows the rest of them to enter low-power states and save power. Additionally, if CPUs can be idled in larger groups (all CPUs on a socket, for example), less power is used. If an idle CPU in a low-power state is selected to run a waking task, an increased cost is incurred as the CPU transitions to a higher state.

The scheduler guesses which type of workload is running based on the wakeup pattern, and uses that to decide whether the tasks should be grouped closely together (for better cache utilization and power consumption), or spread widely across the system (for better CPU utilization).

This is where `wake_wide()` comes into the picture. The `wake_wide()` function is one of the scheduler's more intricate heuristics. It determines whether a task that's being woken up should be pulled to a CPU near the task doing the waking, or allowed to run on a distant CPU, potentially on a different NUMA node. The tradeoff is that packing tasks improves cache locality but also increases the chances of overloading CPUs and causing scheduler run-queue latency.

#### History

The current `wake_wide()` functionality was [introduced](<https://marc.info/?l=linux-kernel&m=143688840122477>) by Mike Galbraith in 2015 based on a [problem statement](<https://marc.info/?l=linux-kernel&m=143276177321620>) in a patch from Josef Bacik, who explained that Facebook has a latency-sensitive, heavily multithreaded application that follows the producer-consumer model, with one master task per NUMA node. The application's performance drops dramatically if tasks are placed onto busy CPUs when woken, which happens if the scheduler tries to pack tasks onto neighboring CPUs; cache locality isn't the most important concern for this application, finding an idle CPU is.

So Galbraith created a switching heuristic that counts the number of wakeups between tasks (called "flips") to dynamically identify master/slave relationships. This heuristic, [implemented in `wake_wide()`](<http://elixir.free-electrons.com/linux/v4.12/source/kernel/sched/fair.c#L5355>), feeds into `select_task_rq_fair()` and guides its understanding of the best CPU to put a waking task on. This function is short enough to show directly: 

```
static int wake_wide(struct task_struct *p)
        {
    	unsigned int master = current->wakee_flips;
    	unsigned int slave = p->wakee_flips;
    	int factor = this_cpu_read(sd_llc_size);
    
    	if (master < slave)
    		swap(master, slave);
    	if (slave < factor || master < slave * factor)
    		return 0;
    	return 1;
        }
```

If the number of slave tasks is less than the number of CPUs that share a last-level cache (LLC) `wake_wide()` will return zero to indicate that the task should not wake on a different LLC domain. In response, [`select_task_rq_fair()`](<http://elixir.free-electrons.com/linux/v4.12/source/kernel/sched/fair.c#L5964>) will pack the tasks, only looking for an idle CPU within a single LLC domain.

If there are more tasks than CPUs (or no master-slave relationship is detected), then tasks are allowed to spread out to other LLC domains and a more time-consuming system-wide search for an idle CPU is performed. When selecting an idle CPU in a different LLC domain, the current power state impacts the scheduler's choice. Since exiting low-power states takes time, the idle CPU in the highest power state is picked to minimize wakeup latency..

#### A new direction?

Recently, Joel Fernandes [raised some questions](<https://marc.info/?l=linux-kernel&m=149878198008808>) about the `wake_wide()` design, saying: ""I didn't follow why we multiply the slave's flips with llc_size"". Bacik [responded](<https://marc.info/?l=linux-kernel&m=149878376809128>), saying that the current code may try too hard to pack tasks, especially when those tasks don't benefit from the shared LLC: ""I'm skeptical of the slave < factor test, I think it's too high of a bar in the case where cache locality doesn't really matter"". He also suggested that removing the expression altogether might fix the aggressive packing problem.

I [provided some data](<https://marc.info/?l=linux-kernel&m=149882829819900>) to show that dropping the `slave < factor` test can improve the performance of [hackbench](<http://people.redhat.com/mingo/cfs-scheduler/tools/hackbench.c>) by reducing the maximum duration over multiple runs. The reason is related to the example that Bacik described where tasks are packed too aggressively. The tasks in hackbench are not paired in a single reader/writer relationship; instead, all tasks communicate among themselves. If hackbench forks more tasks than can fit in a single LLC domain, the tasks will likely be evenly distributed across multiple LLC domains when the benchmark starts. Subsequent packing by the scheduler causes them to be ping-ponged back and forth across the LLC domains, resulting in extremely poor cache usage, and correspondingly poor performance.

Galbraith was quick to warn against making rash changes to `wake_wide()`: ""If you have ideas to improve or replace that heuristic, by all means go for it, just make damn sure it's dirt cheap. Heuristics all suck one way or another, problem is that nasty old ‘perfect is the enemy of good' adage"". But Bacik continued to push for fully utilizing the entire system's CPUs and tweaking the scheduler to be less eager to pack tasks to a single LLC domain. He suspects that the latencies he sees with Facebook's workload would be reduced if a system-wide search was performed in addition to the single LLC domain search when no idle CPU was found.

One point of view missing from the discussions was the developers who are concerned with power first and performance second. Changing the `wake_wide()` heuristic to pack tasks less aggressively has the potential to cause power consumption regressions.

#### Back to the drawing board

In the end, no proposal was the clear winner. ""I think messing with `wake_wide()` itself is too big of a hammer, we probably need a middle ground"", Bacik said. More testing and analysis will need to be done, but even then, a solution might never appear. The multitude of [available scheduler benchmarks](<https://lwn.net/Articles/725238/>) and tracing tools make analyzing the current behavior the easy part; inventing a solution that improves all workloads at the same time is the real challenge.
