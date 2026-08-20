---
title: "ktask: optimizing CPU-intensive kernel work"
url: https://lwn.net/Articles/771169/
date: "November 9, 2018"
category: Parallel execution; ktask
author: "By Jonathan Corbet November 9, 2018"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jonathan Corbet**  
November 9, 2018

As a general rule, the kernel is supposed to use the least amount of CPU time possible; any time taken by the kernel is not available for the applications the user actually wants to run. As a result, not a lot of thought has gone into optimizing the execution of kernel-side work requiring large amounts of CPU. But the kernel does occasionally have to take on CPU-intensive tasks, such as the initialization of the large amounts of memory found on current systems. The [ktask subsystem](<https://lwn.net/ml/linux-kernel/20181105165558.11698-1-daniel.m.jordan@oracle.com/>) posted by Daniel Jordan is an attempt to improve how the kernel handles such jobs. 

If one is going to try to optimize CPU-intensive work in the kernel, there are a number of constraints that must be met. Obviously, that work should be done as quickly and efficiently as possible; that means parallelizing it across the multiple CPUs found in most current systems. But this work needs to not interfere with the rest of the system, and it should not thwart efforts to reduce power consumption. The current patch set tries to meet those goals, though some parts of the problem have been deferred until later. 

#### Basic usage

To use the ktask subsystem, kernel code must provide two fundamental pieces: a structure describing the work to be done and a "thread function" that can be called to do one sub-portion of the total job. The control structure looks like this: 

```
struct ktask_ctl {
    	ktask_thread_func	kc_thread_func;
    	ktask_undo_func		kc_undo_func;
    	void			*kc_func_arg;
    	size_t			kc_min_chunk_size;
    
    	/* Optional, can set with ktask_ctl_set_*.  Defaults on the right. */
    	ktask_iter_func		kc_iter_func;    /* ktask_iter_range */
    	size_t			kc_max_threads;  /* 0 (uses internal limit) */
        };
```

The first member (`kc_thread_func()`) is the function to do a part of the work, while `kc_func_arg` is private data to be passed to that function. `kc_min_chunk_size` defines the smallest piece of the job that can be passed to a call to `kc_thread_func()`; if the job were to clear a large number of pages of memory, for example, the minimum size might be set to the size of a single page. The other fields will be described below. 

In the usual kernel style, this structure can be initialized in either of two ways (using macros): 

```
DEFINE_KTASK_CTL(name, thread_func, func_arg, min_size);
        struct ktask_ctl ctl = KTASK_CTL_INITIALIZER(thread_func, func_arg, min_size);
```

With that in place, the job can be run with: 

```
int ktask_run(void *start, size_t task_size, struct ktask_ctl *ctl);
```

Here, `start` describes the starting point of the job to be done in a way that the thread function understands. It is mostly opaque to ktask itself, though, for the purpose of splitting the job into pieces, ktask treats `start` as a `char *` pointer by default. The size of the task (in whatever units make sense to the thread function) is given by `task_size`, and `ctl` is the control structure. 

This call will break down the given task into units of at least the specified minimum size and pass pieces of it to the thread function, which has this prototype: 

```
typedef int (*ktask_thread_func)(void *start, void *end, void *arg);
```

The portion of the job to be done is described by `start` and `end`, while `arg` is the `kc_func_arg` value from the control structure. It should return `KTASK_RETURN_SUCCESS` (which happens to be zero) if all went well, or an error code otherwise. 

The call to `ktask_run()` will not return until either the entire job is done or a call to the thread function returns an error. Multiple calls may be run in parallel on different CPUs though, by default, `ktask_run()` will limit itself to CPUs on the current NUMA node. The final return value is, again, either `KTASK_RETURN_SUCCESS` or an error code. 

While running on the local NUMA node is a good default, it will often make sense to spread the work out across multiple nodes. To return to the memory-initialization example, the optimal arrangement would be to have each node initialize the memory that is local to it. If a ktask user needs explicit control over the node that a specific piece of the job should be run on, it starts by creating an array of one or more `ktask_node` structures: 

```
struct ktask_node {
    	void		*kn_start;
    	size_t		kn_task_size;
    	int		kn_nid;
        };
```

The `kn_start` and `kn_task_size` members describe the job in the same way as the `start` and `task_size` arguments to `ktask_run()`. The node to run the job on is stored in `kn_nid`; that value can also be `NUMA_NO_NODE` to allow the job to run on any node in the system. The job is then run with: 

```
int ktask_run_numa(struct ktask_node *nodes, size_t nr_nodes,
    		       struct ktask_ctl *ctl);
```

This call will act like `ktask_run()`, except that it will split it across NUMA nodes as directed by the structures in the `nodes` array. 

#### Advanced details

In the default mode, `ktask_run()` and `ktask_run_numa()` will simply stop if the thread function returns an error. But in some cases there can be cleanup to do if things fail partway through; that has to be managed by ktask, since it holds the knowledge of what part of the job had been completed before the error happened. If the caller provides an "undo function" (as `kc_undo_func` in the control structure), that function will be called on the chunks of the job that had been successfully executed before the error happened. The undo function is not allowed to fail. 

By default, the `start` and `end` values used to define a portion of the job are treated as `char` pointers, and the calculation of chunk sizes is done with simple pointer arithmetic. Callers that need a different interpretation can have it, though, by setting a new "iter function" in the control structure. The default function is: 

```
void *ktask_iter_range(void *position, size_t size)
        {
    	return (char *)position + size;
        }
```

Users can replace it by defining a new function and storing it into the control structure with: 

```
void ktask_ctl_set_iter_func(struct ktask_ctl *ctl, ktask_iter_func (*iter_func));
```

The normal usage of this function would be to use a different pointer type for the `position` and scale the `size` accordingly. 

By default, ktask will run parallel calls to the thread function in a maximum of four threads. That value can be changed with a call to: 

```
void ktask_ctl_set_max_threads(struct ktask_ctl *ctl, size_t max_threads);
```

The number of threads actually used may fall short of the given `max_threads` depending on the nature of the job and the system it's running on. 

#### Performance

Ktask can clearly get a job done more quickly if it is able to spread that job out across multiple idle CPUs in the system. If that work then prevents those CPUs from doing anything else, though, the end result may not look like a net win to the user. To avoid interference with real work, ktask runs its worker threads at the lowest priority available (though still above `SCHED_BATCH`). That, naturally, leads to another problem: what if the system is overloaded and the thread functions never get to run? To avoid that problem, ktask will raise one thread at a time to its priority to allow things to continue at the single-threaded pace, at least. 

The [documentation](<https://lwn.net/ml/linux-kernel/20181105165558.11698-2-daniel.m.jordan@oracle.com/>) claims that ktask will disable itself if the system is running in a power-saving mode. That same documentation also says: ""TODO: Implement this"". While it is agreed that ktask should not drive up power consumption on systems where power is at a premium, there is not yet agreement on how that policy should be implemented. Control-group awareness is another detail that has not yet been worked out. 

Perhaps the simplest example use of ktask can be found in [this patch](<https://lwn.net/ml/linux-kernel/20181105165558.11698-13-daniel.m.jordan@oracle.com/>), which converts `clear_gigantic_page()` (which is tasked with zeroing a 1GB huge page). According to the changelog, if ktask is allowed to use eight threads for this job, it will speed it up by a factor of just over eight — a benefit of being able to use more memory bandwidth overall by spreading the job across the system. Other tasks converted in the patch set (almost all associated with memory initialization in one way or another) show similar improvements. 

This patch set has been simmering on the mailing lists for some time; the [first version](<https://lwn.net/Articles/728023/>) was posted in July 2017. The current revision is the fourth, and it appears to be getting closer to being ready to go upstream. There are still some outstanding issues, though, including the loose ends described above, so it seems likely that at least one more posting will be required. There is also the question of how ktask relates to [padata](<https://lwn.net/Articles/382257/>); Jordan had [not heard of it](<https://lwn.net/ml/linux-kernel/20181106203411.pdce6tgs7dncwflh@ca-dmjordan1.us.oracle.com/>) before being asked, but thinks that its requirements are significantly different from those of ktask. All told, ktask may be fast, but its path into the kernel is a bit less so.
