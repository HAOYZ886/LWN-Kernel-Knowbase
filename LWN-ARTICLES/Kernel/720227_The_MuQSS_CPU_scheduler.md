---
title: The MuQSS CPU scheduler
url: https://lwn.net/Articles/720227/
date: "April 20, 2017"
category: Scheduler
author: "April 20, 2017 This article was contributed by Nur Hussein"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

April 20, 2017

This article was contributed by Nur Hussein

The scheduler is a topic of keen interest for the desktop user; the scheduling algorithm partially determines the responsiveness of the Linux desktop as a whole. Con Kolivas maintains a series of scheduler patch sets that he has tuned considerably over the years for his own use, focusing primarily on latency reduction for a better desktop experience. [In early October 2016](<http://ck-hack.blogspot.my/2016/10/muqss-multiple-queue-skiplist-scheduler.html>), Kolivas updated the design of his popular desktop scheduler patch set, which he renamed MuQSS. It is an update (and a name change) from his previous scheduler, BFS, and it is designed to address scalability concerns that BFS had with an increasing number of CPUs.

#### BFS

To understand the updates to the scheduler in MuQSS, we first take a look at the design of BFS, which is the basis for the new patch set. First released in 2009, BFS was a simplified scheduler that was made in response to Kolivas's issues with the mainline scheduler for desktop use. The mainline scheduler has to work well for CPU-bound tasks as well as keeping the desktop smooth and snappy, and it is a difficult balancing act to achieve. Kolivas does not believe in a one-size-fits-all scheduler, so he crafted his own specifically for desktop kernels.

BFS implements a single, global queue of tasks. There is no priority modification based on sleep behavior, and all priorities are set according to the relevant process's nice value. Kolivas argues that any kind of interactivity estimation algorithm just doesn't work well enough and triggers a lot of false positives when trying to find which tasks are interactive. The single run queue design was chosen because Kolivas wanted a scheduler that made global decisions independently of the originating CPU of the processes on the system, and to avoid cycling through a number of different queues to find the next task to run.

BFS uses an earliest eligible virtual deadline first (EEVDF) algorithm to decide which tasks get to run when. Every task entering the run queue gets a time slice and a deadline. The time slice (`rr_interval`), determines the amount of CPU time every task receives. The deadline is computed as a function of current time plus the product of `rr_interval` and the priority level, which is the nice level in ratio form. Processes with a lower nice value (hence higher priority) get deadlines that are sooner than lower-priority ones. Therefore a high-priority task may get to run many times before a lower priority task's deadline is reached. When a task's time slice expires, it will be requeued with a new deadline. However if the task sleeps or is preempted by a higher-priority task, its deadline is not readjusted; instead, the task is just put back on the queue where it will be attended to again in the next round of scheduling.

The `rr_interval` tunable is set at 6ms by default. The reason for this is the threshold for human perception of jitter is just under 7ms. Kolivas predicts a common case of between zero and two running tasks per CPU and, in such a scenario, a task will need to wait no longer than 6ms to get service. 

BFS does away with explicit CPU load rebalancing algorithms between CPUs, instead opting for selecting the CPU for task execution when scheduling every process that wakes up. The global run queue will select an available idle CPU for the next task that is ready to run. When selecting a CPU, the scheduler will try to select the last CPU the task was running on. Failing that, the CPU selection moves "outward", next trying any hyperthreaded or core siblings that share cache. CPUs in different sockets or on different NUMA nodes are treated as "cache distant", and are the least preferred when selecting a CPU for a task to run on.

Other simplifications in the design of BFS include the elimination of most of the tunable knobs and the lack of support for control groups. Kolivas argues that these are features desktop users have no interest in, and an excess of tuning knobs just creates confusion.

#### MuQSS

Due to BFS's single run queue of tasks across all CPUs with a global lock, Kolivas reports that it suffers from lock contention when the number of CPUs increased beyond 16. Every time the scheduler wanted to make a decision on which task should go next, it needed to scan the entire run queue for one with the earliest deadline. Also, iterating over a linked list led to cache-thrashing behavior.

MuQSS is BFS with multiple run queues, one per CPU. Instead of linked lists, the queues have been implemented as [skip lists](<https://en.wikipedia.org/wiki/Skip_list>). First described by William Pugh in 1990, skip lists are probabilistic data structures that give some of the performance advantages of a balanced binary tree (such as the red-black tree used in [CFS](<https://lwn.net/Articles/230574/>)), but with an implementation that is computationally less expensive — if less deterministically efficient. Kolivas's implementation is a custom skip list for his scheduler where the "levels" are static eight-entry arrays. The size of the array was chosen to be exactly one cache line wide.

The deadlines for MuQSS now use "niffies" for keeping track of time. Niffies are a nanosecond-resolution monotonic counter, the value of which is obtained from the high resolution TSC timers. Niffies were introduced sometime in 2010 for BFS, which initially used jiffies for calculating the deadline. The virtual deadline algorithm is essentially the same as BFS:

```
virtual_deadline = niffies + (prio_ratio * rr_interval)
```

Again, `prio_ratio` is the nice level of a task normalized as a ratio and `rr_interval` is the time slice that the task gets at its nice level. When a task is inserted into the skip list queue, it is ordered by the value of its virtual deadline. As a result, the scheduler can find the next eligible task to run in O(1) time, as that task will be at the head of the queue. Insertion into the skip list is done in O(log n) time. 

When selecting a task to run, the scheduler will do a lockless examination of every CPU's run queue to find an eligible task to run, picking the one with the nearest deadline. The scheduler will use a non-blocking "trylock" attempt when popping the chosen task from the relevant run queue, but will simply move on to the next-nearest deadline on another queue if it fails to acquire the queue lock. Although this means the scheduler sometimes doesn't get to run the task with the absolutely nearest deadline all the time, it does alleviate the problem of lock contention among different CPU queues. 

MuQSS, like BFS, is aware of the topology of the logical CPUs, whether they are hyperthreaded siblings, cores sharing a package, or NUMA. The hyperthreading awareness of the scheduler means that an idle core, if available, will be selected before a hyperthreaded sibling CPU to avoid slowdowns due to the CPU's processing capacity being shared between tasks on on the same core. When two tasks are executing on a hyperthreaded core, the lower-priority task will have its time limited to allow the higher-priority task to get more CPU time. Also included is a feature called "SMT Nice", which effectively proportions CPU time on hyperthreaded siblings to reflect their nice priorities.

BFS (and subsequently, MuQSS) introduced two new scheduler policies called `SCHED_ISO` and `SCHED_IDLEPRIO`. `SCHED_ISO`, or isochronous scheduling, is a policy that allows unprivileged processes to gain quasi-realtime behavior. A task set to `SCHED_ISO` will preempt `SCHED_NORMAL` tasks as long as said `SCHED_ISO` task does not exceed the default CPU usage of 70% (this is a tunable value) over a rolling average of five seconds. If more than one `SCHED_ISO` task is running at the same time, they will execute round-robin. This is to prevent other tasks from starving. If a `SCHED_ISO` task exceeds the 70% threshold it is demoted back temporarily to `SCHED_NORMAL` and is appropriately scheduled until enough time has elapsed that the average CPU usage of the task dips below 70% again. Since the 70% threshold counts for all available CPUs on a system, it is possible to entirely use an available CPU for `SCHED_ISO` tasks on SMP machines. For example, a single `SCHED_ISO` task on a dual-core machine can, at most, only consume 50% of total CPU capacity.

`SCHED_IDLEPRIO` is the opposite of `SCHED_ISO` in that it forces tasks to be ultra-low priority. Although it is similar to `SCHED_IDLE` in the mainline scheduler, there is subtle difference: the mainline scheduler will still eventually run `SCHED_IDLE` tasks even if there are other, higher priority tasks running at the same time, but `SCHED_IDLEPRIO` tasks will only run when absolutely nothing else is demanding the CPU's attention. This is useful for batch tasks that scavenge for idle CPUs such as SETI@Home. To avoid deadlocks, if a `SCHED_IDLEPRIO` task is holding a shared resource, such as a mutex or semaphore, it will be promoted back to `SCHED_NORMAL` temporarily to allow it to run even if there are other higher-priority processes in the queue.

There is a limited set of tunables which can control the behavior of the scheduler: 

  * `rr_interval` which is the CPU quantum, which defaults to 6ms.
  * `interactive`, a tunable to toggle the deadline behavior. If disabled, searching for the next task to run is done independently on each CPU, instead of across all CPUs.
  * `iso_cpu`, which determines what percentage of CPU time, across a rolling five-second average, that isochronous tasks will be allowed to use.

#### Comparison to the mainline kernel

Kolivas has not tried to merge his scheduler into the mainline kernel, and it is unlikely he will try to as his scheduler patches were done with his own desktop use case in mind. However, his patch set has a following amongst some Linux desktop users who report a "smoother" desktop experience from using it. Previous throughput benchmarking efforts comparing BFS to the mainline CFS did not conclusively demonstrate improvements in Kolivas's patch sets one way or another. The MuQSS scheduler has reportedly better [Interbench](<http://ck-hack.blogspot.my/2016/10/interbench-benchmarks-for-muqss-116.html>) benchmark scores than CFS. However, ultimately, it is hard to quantify "smoothness" and "responsiveness" and turn them into an automated benchmark, so the best way for interested users to evaluate MuQSS is to [try it out themselves](<http://ck.kolivas.org/patches/muqss/>).

[I would like to thank Con Kolivas for his help in answering questions about MuQSS.]
