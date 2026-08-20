---
title: "Deadline scheduling: coming soon?"
url: https://lwn.net/Articles/575497/
date: "December 4, 2013"
category: "Scheduler-Deadline scheduling"
author: "By Jonathan Corbet December 4, 2013"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
December 4, 2013

Deadline scheduling was first [covered here](<https://lwn.net/Articles/356576/>) in 2009. Like much of the code in the realtime tree, though, deadline scheduling appears not to be subject to deadlines when it comes to being merged into the mainline. That said, it seems entirely possible that this longstanding project will land in a stable kernel release fairly soon, so a look at the status of this patch set, and the proposed ABI in particular, seems in order. 

To recap briefly: deadline scheduling does away with the concept of process priorities that has been at the core of most CPU scheduler algorithms. Instead, each process provides three parameters to the scheduler: a "worst-case execution time" describing a maximum amount of CPU time needed to accomplish its task, a period describing how often the task must be performed, and a deadline specifying when the task must first be completed. The actual scheduling algorithm is then relatively simple: the task whose deadline is closest runs first. If the scheduler takes care to not allow the creation of deadline tasks when the sum of the worst-case execution times would exceed the amount of available CPU time, it can guarantee that every task will be able to finish by its deadline. 

Deadline scheduling is thus useful for realtime tasks, where completion by a deadline is a key requirement. It is also applicable to periodic tasks like streaming media processing. 

In recent times, work on deadline scheduling has been done by Juri Lelli. He has posted several versions, improving things along the way. His [v8 posting](<https://lwn.net/Articles/570293/>) in October generated a fair amount of discussion, including suggestions from scheduler maintainers [Peter Zijlstra](<https://lwn.net/Articles/575502/>) and [Ingo Molnar](<https://lwn.net/Articles/575503/>) that the time had come to merge this code. That merging did not happen for 3.13, but chances are that it will for a near-future kernel release, barring some sort of unexpected roadblock. The main thing that remains to be done is to get the user-space ABI nailed down, since that aspect is hard to change after it has been released in a mainline kernel. 

#### Controlling the scheduler

To be able to guarantee that deadlines will be met, a deadline scheduler must have an incontestable claim to the CPU, so deadline tasks will run ahead of all other tasks — even those in the realtime scheduler classes. Deadline-scheduled processes cannot take _all_ of the available CPU time, though; the amount of time actually available is controlled by a set of sysctl knobs found under `/proc/sys/kernel/`. The first two already exist in current kernels: `sched_rt_runtime_us` and `sched_rt_period_us`. The first specifies the amount of CPU time (in microseconds) available to realtime tasks, while the second gives the period over which that CPU time is available. By default, 95% of the total CPU time is made available to realtime processes, leaving 5% to give a desperate system administrator a chance to recover a system from a runaway realtime process. 

The new `sched_dl_runtime_us` knob is used to say how much of the realtime allocation is available for use by the deadline scheduler. The default setting allocates 40% for deadline scheduling, but a system administrator may well want to tweak that value. Note that, while realtime scheduling is supported by control groups, deadline scheduling has not yet been implemented at that level. How deadline scheduling should interact with group scheduling raises some interesting questions that have not yet been fully answered. 

The other piece of the ABI allows processes to enter and control the deadline scheduling regime. The current system call for changing a process's scheduling class is: 

```
int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
```

The `sched_param` structure used in this system call is quite simple: 

```
struct sched_param {
    	int sched_priority;
        };
```

So `sched_setscheduler()` works fine for the currently available scheduling classes; the desired class is specified with the `policy` parameter, while the associated process priority goes into `param`. But `struct sched_param` clearly does not have the space needed to hold the three parameters needed with deadline scheduling, and its definition cannot be changed without breaking the existing ABI. So a new system call will be needed. As of this writing the details are still under discussion, but the new ABI can be expected to look something like this: 

```
struct sched_attr {
     	int sched_priority;
     	unsigned int sched_flags;
     	u64 sched_runtime;
     	u64 sched_deadline;
     	u64 sched_period;
    	u32 size;
        };
    
        int sched_setscheduler2(pid_t pid, int policy, const struct sched_attr *param);
        int sched_setattr(pid_t pid, const struct sched_attr *param);
        int sched_getattr(pid_t pid, struct sched_attr *param, unsigned int size);
```

Where `size` (as both a parameter and a structure field) is the size of the `sched_attr` structure. If, in the future, the need arises to add more fields to that structure, the kernel will be able to use the `size` value to determine which version of the structure an application is using and respond accordingly. For the curious: `size` is meant to be specified within `struct sched_attr` when that structure is, itself, an input parameter to the kernel; otherwise `size` is given separately. The `sched_flags` field of `struct sched_attr` is not used in the current version of the patch. 

One other noteworthy detail is that processes running in the new `SCHED_DEADLINE` class are not allowed to fork children in the same class. As with the realtime scheduling classes, this restriction can be worked around by setting the scheduling class to `SCHED_DEADLINE|SCHED_RESET_ON_FORK`, which causes the child to be placed back into the default scheduling class. Without that flag, a call to `fork()` will fail. 

#### Time to merge?

The deadline scheduling patch set has a number of loose ends left to be dealt with, many of which are indicated in the patches themselves. But there comes a point where it is best to go ahead and get the code into the mainline so that said loose ends can be tied down more quickly; the deadline scheduling patches may well have reached that point. Since deadline scheduling can be added without much risk of regressions on systems where it is not in use, there should not be a whole lot more that needs to be dealt with before it can be merged. 

...except, maybe, for one little thing. When deadline scheduling was [discussed at the 2010 Kernel Summit](<https://lwn.net/Articles/412745/>), Linus and others clearly worried that there may not be actual users for this functionality. There has not been a whole lot of effort put into demonstrating users for deadline scheduling since then, though it is worth noting that the Yocto project has included the patch in some of its kernels. The [JUNIPER project](<http://www.juniper-project.org/page/overview>) is also planning to use deadline scheduling, and has been supporting its development. Users like these will definitely help the deadline scheduler's case; Linus has become wary of adding features that may not actually be used. If that little point can be adequately addressed, we may have deadline scheduling in the mainline in the near future.
