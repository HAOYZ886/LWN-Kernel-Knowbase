---
title: Dueling performance monitors
url: https://lwn.net/Articles/310260/
date: "December 9, 2008"
category: Performance monitoring
author: "By Jonathan Corbet December 9, 2008"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
December 9, 2008

Low-level optimization of performance-critical code can be a challenging task. At this point, one assumes, the potential for algorithmic improvements in the targeted code has been realized; what is left is trying to locate and address problems like cache misses, mis-predicted branches, and so on. Such problems can be impossible to find by just looking at the code; one needs support from the hardware. The good news is that contemporary hardware provides that support; most processors can collect a wide range of performance data for analysis. The bad news is that, despite the fact that processors have been able to collect that data for many years, there has never been support for this kind of performance monitoring in the mainline kernel. That situation may be about to change, but, first, the development community will have to make a choice between a venerable out-of-tree implementation and an unexpected competitor. 

The "perfmon" patch set has been under development for some years, but, for a number of reasons, it has never found its way into the mainline kernel. The most recent version of the patch was [posted for review](<http://lwn.net/Articles/309008/>) by Stephane Eranian in late November. The perfmon patches show the signs of all those years of development work and usage experience; they offer a wide set of features and extensive user-space support. The full perfmon patch adds twelve system calls to the kernel; the posted version, though, trims that count back to five in the hope that a narrower interface will have a better chance of getting into the mainline. The additional system calls, one assumes, will be proposed for inclusion sometime after the perfmon core is merged. The reduced interface is described [in the patch set](<https://lwn.net/Articles/310263/>); briefly, an application hooks into the performance monitoring subsystem with a call to: 

```
int pfm_create(int flags, pfarg_sinfo_t *regs);
```

This system call returns a file descriptor to identify the performance monitoring session. The `regs` parameter is used to return a list of performance monitoring registers available on the current system; `flags` is currently unused. 

Specific performance counter registers can be manipulated with: 

```
int pfm_write(int fd, int flags, int type, void *d, size_t sz);
        int pfm_read(int fd, int flags, int type, void *d, size_t sz);
```

These system calls can be used to write values into registers (thus programming the performance monitoring hardware) and to read counter and configuration information from those registers. 

Actually doing some performance monitoring requires a couple more calls: 

```
int pfm_attach(int fd, int flags, int target);
        int pfm_set_state(int fd, int flags, int state);
```

A call to `pfm_attach()` specifies which process is to be monitored; `pfm_set_state()` then turns monitoring on and off. 

There are a couple of distinctive aspects to the perfmon interface. One is that it knows almost nothing about the specific performance monitoring registers; that information, instead, is expected to live in user space. As a result, the bare perfmon system call interface is probably not something that most monitoring applications would use; instead, those system calls are hidden behind a user-space library which knows how to program different types of processors for the desired results. Beyond that, perfmon uses the `ptrace()` mechanism to stop the monitored process while performance counters are being queried; as a result, the monitoring process must have the right to trace the target process. 

On December 4, Thomas Gleixner and Ingo Molnar posted [a surprise announcement](<http://lwn.net/Articles/310176/>) of a new performance counter subsystem. The announcement states: 

We are aware of the perfmon3 patchset that has been submitted to lkml recently. Our patchset tries to achieve a similar end result, with a fundamentally different (and we believe, superior :-) design. 

This is not the first time that these developers have shown up with an out-of-the-blue reimplementation of somebody else's subsystem; other examples include the CFS scheduler, high-resolution timers, dynamic tick, and realtime preemption. Most of the time, the new code quickly supplants the older version - an occurrence which is not always pleasing to the original developers - but the situation does not seem quite as straightforward this time. 

The [proposed interface](<https://lwn.net/Articles/310269/>) is much simpler, adding a single system call: 

```
int perf_counter_open(u32 hw_event_type, u32 hw_event_period,
                              u32 record_type, pid_t pid, int cpu);
```

This call will return a file descriptor corresponding to a single hardware counter. A call to `read()` will then return the current value of the counter. The `hw_event_period` can be used to block reads until the counter overflows the given value, allowing, for example, events to be queried in batches of 1000. The `pid` parameter can be used to target a specific process, and `cpu` can restrict monitoring to a specific processor. 

There are a few advantages claimed for the new implementation. The simplicity of the system call interface is one of those; it is possible to write a very simple application to perform monitoring tasks, with no additional libraries required. The [second version of the patch](<http://lwn.net/Articles/310150/>) includes a simple "kerneltop" utility which can display a constantly-updated profile of anything the performance counting hardware can monitor. Another advantage is the avoidance of `ptrace()`; this reduces the amount of privilege needed by the monitoring process and avoids perturbing the monitored process by stopping and restarting it. The management of counters is said to be more flexible, with facilities for sharing counters between processes and reserving them for administrative access. The low-level hardware interface is said to be simpler as well. 

Those claimed advantages notwithstanding, a number of complaints have been raised with regard to the new performance monitoring code. Two of those seem to be at the top of the list: the single counter per file descriptor API, and programming the hardware performance monitoring unit inside the kernel. On the API side, the biggest concern is that putting each counter behind its own file descriptor makes it very hard to correlate two or more counters. Reading two counters requires two independent `read()` system calls; as is always the case, just about anything could happen between those two calls. So it's hard to tell how two different counter values relate to each other. But that sort of correlation is exactly what developers doing performance optimization want to do. Paul Mackerras [says](<https://lwn.net/Articles/310273/>): 

Your API has as its central abstraction the "counter". I am saying that that is the wrong abstraction. The abstraction really needs to be a set of counters that are all active over precisely the same interval, so that their values can be meaningfully compared and related to each other. 

In response, Ingo [argues](<https://lwn.net/Articles/310274/>) that the loss of precision caused by independent `read()` calls is small - much smaller than the muddying of the results caused by stopping the target process so that all of the counters can be read at the same time. That argument does not appear to have convinced the detractors, though. 

The other complaint is that moving the counter programming task into the kernel requires that the kernel know about the complexities of every possible performance monitoring unit it may encounter. This hardware sits at the core of the most performance-critical CPU subsystems, so its design parameters value non-interference above features or a straightforward programming interface. So programming it can be a complex business, involving sizeable tables describing how various operations interact with each other. The perfmon code keeps those tables in a user-space library, but the alternative implementation won't allow that. Quoting Paul again: 

Now, the tables in perfmon's user-land libpfm that describe the mapping from abstract events to event-selector values and the constraints on what events can be counted together come to nearly 29,000 lines of code just for the IBM 64-bit powerpc processors. 

Your API condemns us to adding all that bloat to the kernel, plus the code to use those tables. 

Paul (and others) argue that this information - which can add up to hundreds of kilobytes - is better kept in user space. 

There also seems to be a bit of concern over the fact that Stephane had clearly [never heard about this work](<https://lwn.net/Articles/310276/>) before it was posted for review. It must, indeed, be a shock to work on a subsystem for years, then find a proposed replacement sitting in one's mailbox. As David Miller [put it](<https://lwn.net/Articles/310277/>): 

And also, another part of the backlash is that the poor perfmon3 person was completely blindsided by this new stuff. Which to be honest was pretty unfair. He might have had great ideas about the requirements (even if you don't give a crap about his approach to achieving those requirements) and thus could have helped avoid the past few days of churn. 

So, at this point, what will happen with performance monitoring is unclear at best. Perhaps, though, this discussion will have the effect of raising the profile of performance monitoring, which has been without proper kernel support for many years. The merging of either solution - or, perhaps, a combination of both - seems like it has to be an improvement over having no support at all.
