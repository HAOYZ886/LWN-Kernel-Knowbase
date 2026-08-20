---
title: The slow work mechanism
url: https://lwn.net/Articles/329464/
date: "April 22, 2009"
category: Kernel threads; Slow work
author: "By Jonathan Corbet April 22, 2009"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Jonathan Corbet**  
April 22, 2009

Many years ago, your editor heard Van Jacobson state that naming an algorithm "slow start" was one of the biggest mistakes he had ever made. The name refers to the technique of ramping up transmit rates slowly until the carrying capacity of the connection is determined. But others just saw "slow" and complained that they didn't want their connections to be slow. The fact that "slow start" made the net faster was lost on them. One might wonder if David Howells's "slow work" mechanism - merged for 2.6.30 - could run into similar problems; no kernel developer wants things to run slowly. But, as with slow start, running things slowly is not the point. 

Slow work is a thread pool implementation - yet another thread pool, one might say. The kernel already has [workqueues](<http://lwn.net/Articles/211279/>) and the [asynchronous function call](<http://lwn.net/Articles/314808/>) infrastructure; the distributed storage (DST) module added to the -staging tree for 2.6.30 also has a thread pool hidden within it. Each of these pools is aimed at a different set of uses. Workqueues provide per-CPU threads dedicated to specific subsystems, while asynchronous function calls are optimized for specific ordering of tasks. Slow work, instead, looks like a true "batch job" facility which can be used by kernel subsystems to run tasks which are expected to take a fair amount of time in their execution. 

A kernel subsystem which wants to run slow work jobs must first declare its intention to the slow work code: 

```
#include <linux/slow-work.h>
    
        int slow_work_register_user(void);
```

The call to slow_work_register_user() ensures that the thread pool is set up and ready for work - no threads are created before the first user is registered. The return value will be either zero (on success) or the usual negative error code. 

Actual slow work jobs require the creation of two structures: 

```
struct slow_work;
    
        struct slow_work_ops {
    	int (*get_ref)(struct slow_work *work);
    	void (*put_ref)(struct slow_work *work);
    	void (*execute)(struct slow_work *work);
        };
```

The `slow_work` structure is created by the caller, but is otherwise opaque. The `slow_work_ops` structure, created separately, is where the real work gets done. The `execute()` function will be called by the slow work code to get the actual job done. But first, `get_ref()` will be called to obtain a reference to the `slow_work` structure. Once the work is done, `put_ref()` will be called to return that reference. Slow work items can hang around for some time after they have been submitted, so reference counting is needed to ensure that they are freed at the right time. The implementation of `get_ref()` and `put_ref()` functions is not optional. 

In practice, kernel code using slow work will create its own structure which contains the `slow_work` structure and some sort of reference-counting primitive. The `slow_work` structure must be initialized with one of: 

```
void slow_work_init(struct slow_work *work, const struct slow_work_ops *ops);
        void vslow_work_init(struct slow_work *work, const struct slow_work_ops *ops);
```

The difference between the two is that `vslow_work_init()` identifies the job as "very slow work" which can be expected to run (or sleep) for a significant period of time. The documentation suggests that writing to a file might be "slow work," while "very slow work" might be a sequence of file lookup, creation, and `mkdir()` operations. The slow work code actually prioritizes "very slow work" items over the merely slow ones, but only up to the point where they use 50% (by default) of the available threads. Once the maximum number of very slow jobs is running, only "slow work" tasks will be executed. 

Actually getting a slow work task running is done with: 

```
int slow_work_enqueue(struct slow_work *work);
```

This function queues the task for running. It will succeed unless the associated `get_ref()` function fails, in which case `-EAGAIN` will be returned. 

Slow work tasks can be enqueued multiple times, but no count is kept, so a task enqueued several times before it begins to execute will only run once. A task which is enqueued while it is running is indeed put back on the queue for a second execution later on. The same task is guaranteed to not run on multiple CPUs simultaneously. 

There is no way to remove tasks which have been queued for execution, and there is no way (built into the slow work mechanism) to wait for those tasks to complete. A "wait for completion" functionality can certainly be created by the caller if need be. The general assumption, though, seems to be that slow work items can be outstanding for an indefinite period of time. As long as tasks with a non-zero reference count exist, any resources they depend on need to remain available. 

There are three parameters for controlling slow work which appear under `/proc/sys/kernel/slow-work`: `min-threads` (the minimum size of the thread pool), `max-threads` (the maximum size), and `vslow-percentage` (the maximum percentage of the available threads which can be used for "very slow" tasks). The defaults allow for between two and four threads, 50% of which can run "very slow" tasks. 

The only user of slow work in the 2.6.30 kernel is the FS-Cache file caching subsystem. There is a clear need for thread pool functionality, though, so it would not be surprising to see other users show up in future releases. What might be more surprising (though desirable) would be a consolidation of thread pool implementations in a future development cycle.
