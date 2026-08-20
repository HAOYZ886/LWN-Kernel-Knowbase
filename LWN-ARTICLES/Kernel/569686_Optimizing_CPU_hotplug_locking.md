---
title: Optimizing CPU hotplug locking
url: https://lwn.net/Articles/569686/
date: "October 9, 2013"
category: "Hotplug; Read-copy-update-rcu sync; Scalability"
author: "By Jonathan Corbet October 9, 2013"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
October 9, 2013

The 3.12 development cycle has seen an increased level of activity around scalability and, in particular, the reduction of locking overhead. Traffic on the linux-kernel mailing list suggests that this work will extend into 3.13, if not beyond. One of several patch sets currently under development relates to CPU hotplugging — the process of adding CPUs to (or removing them from) a running system. 

CPU hotplugging adds complications to a number of kernel subsystems; the fact that processors can come and go at arbitrary times must always be taken into account. Needless to say, hotplug operations must be restricted to times when the kernel is prepared for them; to that end, the kernel provides a reference count mechanism to allow any thread to block CPU hotplugging. The reference count is raised with `get_online_cpus()` to indicate that the set of online CPUs should not be changed; the reference count is decremented with `put_online_cpus()`. 

The implementation of `get_online_cpus()` in current kernels is relatively straightforward: 

```
mutex_lock(&cpu_hotplug.lock);
        cpu_hotplug.refcount++;
        mutex_unlock(&cpu_hotplug.lock);
```

Code that is managing an actual hotplug operation will acquire `cpu_hotplug.lock` (after waiting for the reference count to drop to zero if need be) and hold it for the duration of the operation. This mechanism ensures that no thread will see a change in the set of active CPUs while it holds a reference, but there is a bit of a problem: each reference-count change causes the cache line containing the lock and the count to bounce around the system. Since calls to `get_online_cpus()` and `put_online_cpus()` can happen frequently in the core kernel, this bouncing can be hard on performance. 

The really sad fact in this case, though, is that CPU hotplug events are exceedingly rare; chances are that, in most running systems, there will never be a hotplug event until the system shuts down. This kind of pattern argues for a different approach to locking, where the common case is as fast as it can be made to be. That is exactly what [Peter Zijlstra's CPU hotplug locking patch set](<https://lwn.net/Articles/569770/>) sets out to do. To reach that goal, Peter has had to create a custom locking mechanism — a practice which is frowned upon whenever it can be avoided — and incorporate a new RCU-based synchronization mechanism as well. The patch series shows the evolution of this approach; this article will follow in the same path. 

#### The new locking scheme

Peter's patch adds a couple of new variables related to CPU hotplugging: 

  * `__cpuhp_refcount` is the new form of the reference count controlling hotplug operations. Unlike its predecessor, though, it is a per-CPU variable, so each CPU can tweak its own count without causing cache-line contention. 

  * `__cpuhp_state` is an enum with three values: `readers_fast`, `readers_slow`, and `readers_block`. 

"Readers," in the context of this locking mechanism, are threads that call `get_online_cpus()`; they need the set of online CPUs to stay stable but make no changes to it. A "writer," instead, is a thread executing an actual CPU hotplug operation. 

The state starts out as `readers_fast`, an indication that no CPU hotplugging activity is going on and that, thus, readers can take the fast path through the locking code. With that in mind, here is a simplified form of the core of the new `get_online_cpus()`: 

```
if (likely(__cpuhp_state == readers_fast))
    	__this_cpu_inc(__cpuhp_refcount);
        else
    	__get_online_cpus();
```

So, when things are in the `readers_fast` state, `get_online_cpus()` turns into a simple, per-CPU increment operation, with no cache-line contention. Otherwise the slow-path code (found in `__get_online_cpus()`) must be run. The `put_online_cpus()` code looks similar; when no CPUs are coming or going, all that is needed is a per-CPU decrement operation. 

When it is time to add or remove a CPU, the hotplug code will make a call to `cpu_hotplug_begin()`. This function begins with these three lines of code: 

```
__cpuhp_state = readers_slow;
        synchronize_sched();
        __cpuhp_state = readers_block;
```

The assignment to `__cpuhp_state` puts an end to the fast-path reference count operations. A call to `synchronize_sched()` (a read-copy-update primitive that waits for each CPU to schedule at least once) is necessary to ensure that no thread is still running in the hot-path code in either `get_online_cpus()` or `put_online_cpus()`. Once that condition is assured, the state is changed again to `readers_block`. That will cause new readers to block (as described below), but there may still be old readers running, so the `cpu_hotplug_begin()` call will block until all of the per-CPU reference counts fall to zero. 

At this point, it is worth looking at what happens in the `__get_online_cpus()` slow path. If that code sees `__cpuhp_state` as `readers_slow`, it will simply increment the per-CPU reference count and return in the usual manner; it is still possible to obtain a reference in this state. If, instead, it sees `readers_block`, it will increment an (atomic) count of waiting threads, then block on a wait queue without raising the reference count. The `__put_online_cpus()` slow path is simpler: it decrements the reference count as usual, then calls `wake_up()` to wake any thread that might be waiting in `cpu_hotplug_begin()`. 

Returning to that function: `cpu_hotplug_begin()` will return to its caller once all references have been returned (all of the per-CPU reference counts have dropped to zero). At that point, it is safe to carry out the CPU hotplug event, changing the set of online CPUs; afterward, a call is made to `cpu_hotplug_done()`. That function reverses what was done in `cpu_hotplug_begin()` in the following way: 

```
__cpuhp_state = readers_slow;
        wake_up_all(&cpuhp_readers);
        synchronize_sched();
        __cpuhp_state = readers_fast;
```

It will then wait until the count of waiting readers drops to zero before returning. This wait (like the entire hotplug operation) is done holding the global hotplug mutex, so, while that wait is happening, no other CPU hotplug operations can begin. 

This code raises some interesting questions, starting with: why does `cpu_hotplug_done()` have to set the state to `readers_slow`, rather than re-enabling the fast paths immediately? The purpose here is to ensure that any new readers that come along will see all of the changes made by the writer while readers were blocked. The extra memory barriers in the slow path will ensure that all CPUs see the new state of the world correctly. The `synchronize_sched()` call is needed to ensure that any thread that might try to block will have done so; that means, among other things, that the count of waiting readers will be complete. 

Why does `cpu_hotplug_begin()` explicitly block all readers? This behavior turns the CPU hotplug locking mechanism into one that is biased toward writers; the moment a writer comes along, new readers are blocked almost immediately. Things are done this way because there could be a _lot_ of readers in a large and busy system; if they cannot be blocked, writers could be starved indefinitely. Given that CPU hotplug operations are so rare, there should be no real performance issues resulting from blocking readers and allowing hotplug operations to proceed as soon as possible. 

What is the purpose of the count of waiting readers? A single writer can put readers on hold, but those readers should be allowed to proceed before a second hotplug operation can be carried out. By waiting for the count to drop to zero, `cpu_hotplug_done()` ensures that every reader that was blocked will be able to proceed before the next writer clogs up the works again. 

The end result of all this work is that, most of the time, the locking overhead associated with `get_online_cpus()` will be replaced by a fast, per-CPU increment operation. There is a cost paid in the form of more complex locking code and, perhaps, more expensive hotplug operations, but a CPU hotplug event is not something that needs to be optimized for. So it seems like a net win. 

#### rcu_sync

Interestingly enough, though, Peter's patch still wasn't fast enough for some people. In particular, the `synchronize_sched()` calls were seen as being too expensive. To address this problem, Oleg Nesterov put together [a patch](<https://lwn.net/Articles/569779/>) adding a new "rcu_sync" mechanism. In brief, the API looks like: 

```
struct rcu_sync_struct;
    
    	void rcu_sync_enter(struct rcu_sync_struct *rss);
    	void rcu_sync_exit(struct rcu_sync_struct *rss);
    
    	bool rcu_sync_is_idle(struct rcu_sync_struct *rss);
```

An rcu_sync structure starts out in the "idle" state; it can be moved out of that state with one or more `rcu_sync_enter()` calls. When an equal number of `rcu_sync_exit()` calls have been made, the structure will test as idle again. The state changes are made using RCU so that, in particular, `rcu_sync_exit()` works via an ordinary RCU callback rather than calling `synchronize_sched()`. 

To use this infrastructure with CPU hotplugging, Peter defined the "idle" state as meaning that no hotplug operations are underway; then, calls to `rcu_sync_is_idle()` can replace tests against the `readers_fast` state described above — and the `synchronize_sched()` calls as well. That should make things faster — though the extent of the speedup is not entirely clear. 

After all this work is done, a simple mutex-protected reference count has been replaced by a few hundred lines of complex, one-off locking code. In the process, the kernel has gotten a little bit harder to understand. This new complexity is unfortunate, but it seems to be an unavoidable by-product of the push for increased scalability. Getting the best performance out of a highly concurrent system can only be made so simple.
