---
title: "Who let the hogs out?"
url: https://lwn.net/Articles/378859/
date: "March 16, 2010"
category: CPUhog; Scheduler
author: "By Jonathan Corbet March 16, 2010"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 16, 2010

As a normal rule of business, the kernel tries to avoid using more system resources than are absolutely necessary; system time is better spent running user-space programs. So Tejun Heo's [cpuhog patch](<http://lwn.net/Articles/377720/>) may come across as a little surprising; it creates a mechanism by which the kernel can monopolize one or more CPUs with high-priority processes doing nothing. But there is a good reason behind this patch set; it should even improve performance in some situations. 

Suppose you wanted to take over one or more CPUs on the system. The first step is to establish a hog function: 

```
#include <linux/cpuhog.h>
    
        typedef int (*cpuhog_fn_t)(void *arg);
```

When hog time comes, this function will be called at the highest possible priority. If the intent is truly to hog the CPU, the function should probably spin in a tight loop. But one should take care to ensure that this loop will end at some point; one does not normally want to take the CPU out of commission permanently. 

The monopolization of processors is done with any of: 

```
int hog_one_cpu(unsigned int cpu, cpuhog_fn_t fn, void *arg);
        void hog_one_cpu_nowait(unsigned int cpu, cpuhog_fn_t fn, void *arg,
    			    struct cpuhog_work *work_buf);
        int hog_cpus(const struct cpumask *cpumask, cpuhog_fn_t fn, void *arg);
        int try_hog_cpus(const struct cpumask *cpumask, cpuhog_fn_t fn, void *arg);
```

A call to `hog_one_cpu()` will cause the given `fn()` to be run on `cpu` in full hog mode; the calling process will wait until `fn()` returns; at which point the return value from `fn()` will be passed back. Should there be other useful work to do (on a different CPU, one assumes), `hog_one_cpu_nowait()` can be called instead; it will return immediately, while `fn()` may still be running. The `work_buf` structure must be allocated by the caller and be unused, but the caller need not worry about it beyond that. 

Sometimes, total control over one CPU is not enough; in that case, `hog_cpus()` can be called to run `fn()` simultaneously on all CPUs indicated by `cpumask`. The `try_hog_cpus()` variant is similar, but, unlike `hog_cpus()`, it will not wait if somebody else got in and started hogging CPUs first. 

So what might one use this mechanism for? One possibility is `stop_machine()`, which is called to ensure that absolutely nothing of interest is happening anywhere in the system for a while. Calls to `stop_machine()` usually happen when fundamental changes are being made to the system - examples include the insertion of dynamic probes, loading of kernel modules, or the removal of CPUs. It has always worked in the same way as the CPU hog functions do - by running a high-priority thread on each processor. 

The new `stop_machine()` implementation, naturally, uses `hog_cpus()`. Unlike the previous implementation, though (which used workqueues), the new code takes advantage of the CPU hog threads which already exist. That eliminates [a performance bug](<http://thread.gmane.org/gmane.linux.kernel/957726>) reported by Dimitri Sivanich, whereby the amount of time required to boot a system would be doubled by the extra overhead of various `stop_machine()` calls. 

Another use for this facility is to force all CPUs to quickly go through the scheduler; that can be useful if the system wants to force a transition to a new read-copy-update grace period. Formerly, this task was bundled into the migration thread, which already runs on each CPU, in a bit of an awkward way; now it's a straightforward CPU hog call. 

The migration thread itself is also a user of the single-CPU hogging function. This thread comes into play when the system wants to migrate a process which is running on a given CPU. The first thing that needs to happen is to force that process out of the CPU - a job for which the CPU hog is well suited. Once the hog has taken over the CPU, the just-displaced process can be moved to its new home. 

The end result is the removal of a fair amount of code, a cleaned-up migration thread implementation, and improved performance in `stop_machine()`. Some [concerns](<https://lwn.net/Articles/378866/>) were raised that passing a blocking function as a CPU hog could create problems in some situations. But blocking in a CPU hog seems like an inherently contradictory thing to do; one assumes that the usual response will be "don't do that". And, in fact, [version 2 of the patch](<http://lwn.net/Articles/378968/>) disallows sleeping in hog functions. Of course, the "don't do that" response will also apply to most uses of CPU hogs in general; taking over processors in the kernel is still considered to be an antisocial thing to do most of the time.
