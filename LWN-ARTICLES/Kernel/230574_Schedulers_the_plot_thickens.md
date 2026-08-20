---
title: "Schedulers: the plot thickens"
url: https://lwn.net/Articles/230574/
date: "April 17, 2007"
category: "Interactivity; Scheduler-Completely fair scheduler"
author: 
---

The [RSDL scheduler](<http://lwn.net/Articles/224865/>) (since renamed the staircase deadline scheduler) by Con Kolivas was, for a period of time, assumed to be positioned for merging into the mainline, perhaps as soon as 2.6.22. Difficulties with certain workloads made the future of this scheduler a little less certain. Now Con would appear to have rediscovered one of the most reliable ways of getting a new idea into the kernel: post some code then wait for Ingo Molnar to rework the whole thing in a two-day hacking binge. So, while Con has recently [updated the SD scheduler patch](<http://lwn.net/Articles/230500/>), his work now looks like it might be upstaged by Ingo's new [completely fair scheduler](<http://lwn.net/Articles/230501/>) (CFS), at [version 2](<http://lwn.net/Articles/230752/>) as of this writing. 

There are a number of interesting aspects to CFS. To begin with, it does away with the arrays of run queues altogether. Instead, the CFS works with a single [red-black tree](<http://lwn.net/Articles/184495/>) to track all processes which are in a runnable state. The process which pops up at the leftmost node of the tree is the one which is most entitled to run at any given time. So the key to understanding this scheduler is to get a sense for how it calculates the key value used to insert a process into the tree. 

That calculation is reasonably simple. When a task goes into the run queue, the current time is noted. As the process waits for the CPU, the scheduler tracks the amount of processor time it would have been entitled to; this entitlement is simply the wait time divided by the number of running processes (with a correction for different priority values). For all practical purposes, the key is the amount of CPU time due to the process, with higher-priority processes getting a bit of a boost. The short-term priority of a process will thus vary depending on whether it is getting its fair share of the processor or not. 

It is only a slight oversimplification to say that the above discussion covers the entirety of the CFS scheduler. There is no tracking of sleep time, no attempt to identify interactive processes, etc. In a sense, the CFS scheduler even does away with the concept of time slices; it's all a matter of whether a given process is getting the share of the CPU it is entitled to given the number of processes which are trying to run. The CFS scheduler offers a single tunable: a "granularity" value which describes how quickly the scheduler will switch processes in order to maintain fairness. A low granularity gives more frequent switching; this setting translates to lower latency for interactive responses but can lower throughput slightly. Server systems may run better with a higher granularity value. 

Ingo claims that the CFS scheduler provides solid, fair interactive response in almost all situations. There's a whole set of nasty programs in circulation which can be used to destroy interactivity under the current scheduler; none of them, says Ingo, will impact interactivity under CFS. 

The CFS posting came with another feature which surprised almost everybody who has been watching this area of kernel development: a modular scheduler framework. Ingo describes it as "an extensible hierarchy of scheduler modules," but, if so, it's a hierarchy with no branches. It's a simple linked list of modules in priority order; the first scheduler module which can come up with a runnable task gets to decide who goes next. Currently two modules are provided: the CFS scheduler described above and a simplified version of the real-time scheduler. The real-time scheduler appears first in the list, so any real-time tasks will run ahead of normal processes. 

There is a relatively small set of methods implemented by each scheduler module, starting with the queueing functions: 

```
void (*enqueue_task) (struct rq *rq, struct task_struct *p);
        void (*dequeue_task) (struct rq *rq, struct task_struct *p);
        void (*requeue_task) (struct rq *rq, struct task_struct *p);
```

When a task enters the runnable state, the core scheduler will hand it to the appropriate scheduler module with `enqueue_task()`; a task which is no longer runnable is taken out with `dequeue_task()`. The `requeue_task()` function puts the process behind all others at the same priority; it is used to implement `sched_yield()`. 

A few functions exist for helping the scheduler track processes: 

```
void (*task_new) (struct rq *rq, struct task_struct *p);
        void (*task_init) (struct rq *rq, struct task_struct *p);
        void (*task_tick) (struct rq *rq, struct task_struct *p);
```

The core scheduler will call `task_new()` when processes are created. `task_init()` initializes any needed priority calculations and such; it can be called when a process is reniced, for example. The `task_tick()` function is called from the timer tick to update accounting and possibly switch to a different process. 

The core scheduler can ask a scheduler module whether the currently executing process should be preempted now: 

```
void (*check_preempt_curr) (struct rq *rq, struct task_struct *p);
```

In the CFS scheduler, this check tests the given process's priority against that of the currently running process, followed by the fairness test. When the fairness test is done, the scheduling granularity is taken into account, possibly allowing a process to run a little longer than strict fairness would allow. 

When it's time for the core scheduler to choose a process to run, it will use these methods: 

```
struct task_struct * (*pick_next_task) (struct rq *rq);
        void (*put_prev_task) (struct rq *rq, struct task_struct *p);
```

The call to `pick_next_task()` asks a scheduler module to decide which process (among those in the class managed by that module) should be running currently. When a task is switched out of the CPU, the module will be informed with a call to `put_prev_task()`. 

Finally, there's a pair of methods intended to help with load balancing across CPUs: 

```
struct task_struct * (*load_balance_start) (struct rq *rq);
        struct task_struct * (*load_balance_next) (struct rq *rq);
```

These functions implement a simple iterator which the scheduler can used to work through all processes currently managed by the scheduling module. 

One assumes that this framework could be used to implement different scheduling regimes in the future. It might need some filling out; there is, for example, no way to prioritize scheduling modules (or choose the default module) other than changing the source. Beyond that, if anybody ever wants to implement modules which schedule tasks at the same general priority level, the strict priority ordering of the current framework will have to change - and that could be an interesting task. But it's a start. 

The reason that this development is so surprising is that nobody had really been talking about modular schedulers. And the reason for that silence is that pluggable scheduling frameworks had been soundly rejected in the past \- [by Ingo Molnar](<http://lwn.net/Articles/109460/>), among others: 

So i consider scheduler plugins as the STREAMS equivalent of scheduling and i am not very positive about it. Just like STREAMS, i consider 'scheduler plugins' as the easy but deceptive and wrong way out of current problems, which will create much worse problems than the ones it tries to solve. 

So the obvious question was: what has changed? Ingo has posted [an explanation](<https://lwn.net/Articles/230628/>) which goes on at some length. In essence, the previous pluggable scheduler patches were focused on replacing the entire scheduler rather than smaller pieces of it; they did not help to make the scheduler simpler. 

So now there are three scheduler replacement proposals on the table: SD by Con Kolivas, CFS by Ingo Molnar, and "nicksched" by Nick Piggin (a longstanding project which clearly deserves treatment on this page as well). For the moment, Con appears to have decided to take his marbles and go home, removing SD from consideration. Still, there are a few options out there, and one big chance (for now) to replace the core CPU scheduler. While Ingo's work has been generally well received, not even Ingo is likely to get a free pass on a decision like this; expect there to be some serious discussion before an actual replacement of the scheduler is made. Among other things, that suggests that a new scheduler for 2.6.22 is probably not in the cards.
