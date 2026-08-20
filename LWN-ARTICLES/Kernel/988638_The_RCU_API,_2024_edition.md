---
title: "The RCU API, 2024 edition"
url: https://lwn.net/Articles/988638/
date: "September 13, 2024"
category: "Read-copy-update"
author: "September 13, 2024 This article was contributed by Paul McKenney"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

September 13, 2024

This article was contributed by Paul McKenney

Read-copy-update (RCU) is a synchronization mechanism that was added to the Linux kernel in October 2002. RCU is most frequently used as a [replacement for reader-writer locking](<https://linuxfoundation.org/webinars/unraveling-rcu-usage-mysteries/>), but is also used in [a number of other ways](<https://linuxfoundation.org/webinars/unraveling-rcu-usage-mysteries-additional-use-cases/>). This mechanism is notable in that RCU readers do not directly synchronize with RCU updaters, which makes RCU read paths extremely fast and also permits RCU readers to accomplish useful work even when running concurrently with RCU updaters. Those wishing an in-depth introduction to RCU are invited to consult the LWN series [here](<http://lwn.net/Articles/262464/>), [here](<http://lwn.net/Articles/263130/>), and [here](<http://lwn.net/Articles/264090/>). 

This article covers recent changes to the RCU API; it was contributed by Paul McKenney, Boqun Feng, Frederic Weisbecker, Joel Fernandes, Neeraj Upadhyay, and Uladzislau Rezki. 

Although the basic idea behind RCU has not changed during the three decades following its introduction into DYNIX/ptx, the RCU API has evolved significantly since the [2010](<http://lwn.net/Articles/418853/>), [2014](<https://lwn.net/Articles/609904/>), and [2019](<https://lwn.net/Articles/777036/>) editions of the Linux-kernel RCU API. The most recent five years of this evolution is documented by the following sections. 

  1. Summary of RCU API changes
  2. How did those 2019 predictions turn out?
  3. What next for the RCU API?

If that is not enough RCU for you, there are a lot more details to be found in the associated [background-material article](<https://lwn.net/Articles/988641/>). 

But first, a few announcements: 

  1. People familiar with the 2019 edition of the RCU API will note several additional names on the byline. These people have taken up the challenge of learning the various Linux-kernel RCU implementations, each investing significant time over a period of years. Not that Paul intends to go anywhere anytime soon, his father having taken early retirement only recently, but Mother Nature might force him into her retirement program at any time. Paul therefore asks you to please take them seriously, because if something happens to him, they are your Linux-kernel RCU maintainers. 
  2. It is wise to CC `rcu@vger.kernel.org` on any RCU-related email that might otherwise have been sent privately to Paul, who is likely to become less aggressive about checking email during off-hours. 
  3. In a not-unrelated change, the source of truth for RCU commits has moved from Paul's venerable [-rcu tree](<https://git.kernel.org/pub/scm/linux/kernel/git/paulmck/linux-rcu.git>) to a [shared RCU tree](<https://git.kernel.org/pub/scm/linux/kernel/git/rcu/linux.git>). His -rcu tree will continue to exist, but will be one of several feeding into the shared RCU tree and elsewhere. 

###  Summary of RCU API changes

These sections summarize some of the most visible changes to RCU: 

  1. Lazy RCU grace asynchronous periods
  2. Reworking of `kfree_rcu()`
  3. Polled RCU grace periods
  4. Tasks Rude RCU and Tasks Trace RCU
  5. New SRCU read-side critical sections
  6. Read-side guards
  7. RCU callback dynamic (de-)offloading
  8. Miscellaneous

####  Lazy RCU grace asynchronous periods

The addition of lazy RCU grace periods was prompted by energy-efficiency concerns on battery-powered platforms, most notably Android and ChromeOS. This laziness affects callbacks queued via `call_rcu()`, and is not to be confused with the lazy processing of RCU callbacks from `kfree_rcu()`, which is covered in the next section. 

New-age lazy `call_rcu()` grace periods are controlled at build time by a new `CONFIG_RCU_LAZY` kernel configuration option and, in kernels built with `CONFIG_RCU_LAZY=y`, at run time by a new `rcutree.enable_rcu_lazy` kernel boot parameter. There is an additional `CONFIG_RCU_LAZY_DEFAULT_OFF` configuration option that changes the default setting of this new kernel boot parameter to be disabled. In other words, when the kernel boot parameter is not specified, kernels built with `CONFIG_RCU_LAZY=y` but without specifying `CONFIG_RCU_LAZY_DEFAULT_OFF` (or, equivalently, built with `CONFIG_RCU_LAZY_DEFAULT_OFF=n`) will have lazy `call_rcu()` callbacks, while kernels built with `CONFIG_RCU_LAZY=y` and `CONFIG_RCU_LAZY_DEFAULT_OFF=y` will have non-lazy (hurried) `call_rcu()` callbacks. 

The kernel boot parameter overrides the configuration options, so that (for example) booting a `CONFIG_RCU_LAZY=y` kernel with `rcutree.enable_rcu_lazy=1` will result in lazy `call_rcu()` callbacks. Lazy callbacks will wait up to ten seconds before starting a grace period, thus greatly reducing the number of grace periods (and their associated wakeups of idle CPUs) on an almost-idle device. However, `call_rcu()` callbacks will only ever be lazy on `rcu_nocbs` CPUs. The reason for this is that systems with `non-rcu_nocbs` CPUs consume more energy than their `rcu_nocbs` counterparts, so those concerned with battery lifetime would be well-advised to first enable `rcu_nocbs`. This can be done by building the kernel with `CONFIG_RCU_NOCB_CPU=y` and booting it with `rcu_nocbs=all`. 

Testing of lazy `call_rcu()` grace periods showed well in excess of 10% improvements in energy efficiency. 

Some uses of `call_rcu()` cannot tolerate laziness, for example, when the callback function does a wakeup. These uses should instead invoke the new `call_rcu_hurry()` function. Note that invoking `call_rcu_hurry()` on a given CPU will hurry along any earlier lazy callbacks that were previously queued using `call_rcu()`. 

When suspending or hibernating, it is also important for all callbacks to hurry. (But don't take our word for it, ask a user about adding a ten-second delay to suspend!) RCU therefore has an `rcu_pm_notify()` function that hurries callbacks at the start of a suspend or hibernation operation. This same function re-enables laziness when this operation completes. 

####  Reworking of `kfree_rcu()`

From the viewpoint of pre-existing, working code, `kfree_rcu()` works just like it always did. However, under the covers, performance has increased significantly through use of pages of pointers to track the memory and through use of `kfree_bulk()`. Both of these changes greatly improve cache locality, which provides up to a 12% increase in performance. Performance was increased still further for battery-powered devices via carefully developed heuristics that govern how long a partially populated page of pointers waits before being submitted to RCU. 

In addition, `kfree_rcu()` can now handle memory from `kmem_cache_alloc()`, not due to any changes to `kfree_rcu()`, but rather due to changes to `kfree()`. However, please note that `rcu_barrier()` does not wait for memory being freed via `kfree_rcu()`, so that there currently is no way to safely invoke `kmem_cache_destroy()` on module exit if that module ever used `kfree_rcu()` on memory from the corresponding `kmem_cache` structure. A [fix](<https://docs.google.com/document/d/1v0rcZLvvjVGejT3523W0rDy_sLFu2LWc_NR3fQItZaA/edit?usp=sharing>) is in the works. In the meantime, such modules should continue using `call_rcu()` for this use case. 

One disadvantage of `kfree_rcu()` is that an `rcu_head` structure must be embedded within the structure to be freed, costing 16 additional bytes on 64-bit systems. For example, a structure referenced by `p` where the `rcu_head` structure is in a field named `rh` can be RCU-deferred-freed using: 

```
kfree_rcu(&p->rh, rh)
```

This issue is addressed by a new `kfree_rcu_mightsleep()` function, which takes a single pointer to the beginning of the object to be freed, as in `kfree_rcu_mightsleep(p)`, thus avoiding the need for that `rcu_head` structure and its 16 bytes of memory. 

Of course, there is always a catch, and in this case the catch is that, as the name suggests, `kfree_rcu_mightsleep()` might sleep. In fact, if it is unable to allocate the memory needed to track the object being deferred-freed, it will simply invoke `synchronize_rcu()`, which blocks for some tens of milliseconds, and then directly invokes `kfree()`. In contrast, `kfree_rcu()` reacts to low memory by invoking `call_rcu()`, thus giving up cache locality but not caller-visible latency. As always, choose wisely! 

There are versions of the kernel that have a single-argument variant of `kfree_rcu()` instead of `kfree_rcu_mightsleep()`. This single-argument approach proved to be a serious mistake that led to subtle bugs in which people forgot to specify the second argument, which fails (but only sometimes) from atomic contexts such as interrupt handlers. Therefore, recent kernels provide only the double-argument variant of `kfree_rcu()`. Where the old single-argument version would have been used, use `kfree_rcu_mightsleep()` instead. 

There are also `kvfree_rcu()` and `kvfree_rcu_mightsleep()` functions that operate on `vmalloc()` memory. 

In short, if you have RCU callbacks that do nothing but `kfree()`, `vfree()`, or `kmem_cache_free()` to immortal `kmem_cache` structures, these functions might save you both CPU time and a few lines of code. Many of these have already been converted, courtesy of a [Coccinelle](<https://en.wikipedia.org/wiki/Coccinelle_\(software\)>) script and patches from Julia Lawall, but new code is written all the time. 

####  Polled RCU grace periods

The historic RCU API does quite a bit of work for the user, who can simply invoke `synchronize_rcu()` and, upon its return, know that all pre-existing readers are done. Or, if the `synchronize_rcu()` function's blocking is problematic, pass a pointer and a callback function to `call_rcu()` and upon invocation of that callback function, know that all pre-existing readers are done. Better yet, if that callback function is doing nothing but invoking `kfree()` or `kmem_cache_free()`, just pass a pointer and `rcu_head` offset to `kfree_rcu()`, and rely on RCU to do everything, including the freeing. Or even better still, if within a latency-tolerant, non-atomic context, dispense with the `rcu_head` and pass just the pointer itself to `kfree_rcu_mightsleep()`. 

However, as reported [here](<https://lwn.net/Articles/974487/>), there are situations in which these conveniences become counterproductive. For example, in caching situations, it might be that an object that has been queued for deferred free is once again needed. In such cases, it might be helpful to cancel a `synchronize_rcu()`, prevent a `call_rcu()` callback from being invoked, or to prevent an object passed to `kfree_rcu()` or `kfree_rcu_mightsleep()` from being freed. For another example, in situations that invoke many instances of `call_rcu()` in short time periods, RCU's choice of software-interrupt context for callback invocation might be suboptimal. Unfortunately, providing for these situations would further complicate the RCU API and implementation, and would also result in performance degradation. 

**Quick Quiz 1** : Why would `call_rcu()` be helpful in scheduling polling for a particular grace period? 

**(Click for answer)** Because the callback passed to `call_rcu()` tends to be invoked shortly after a grace period has completed, which is an excellent time to do polling for the end of a grace period. Alternatively, if the RCU-callback softirq context is inconvenient, you can instead use `queue_rcu_work()` to schedule a workqueue handler to execute shortly after a grace period completes. 

In recent Linux kernels, RCU instead provides a complete polling API for managing RCU grace periods. This permits RCU users to take full control over all aspects of waiting for grace periods and responding to the end of a particular grace period. For example, the `get_state_synchronize_rcu()` function returns a cookie that can be passed to `poll_state_synchronize_rcu()`. This latter function will return `true` if a grace period has elapsed in the meantime. The user may choose any convenient method to schedule the polling for the end of the grace period, from `mod_timer()` up to and including use of `call_rcu()` itself. Once the grace period in question has elapsed, the user may choose any convenient context from which to free memory, or to undertake whatever other processing is required. 

For a bit more background on RCU's polled grace-period API, please see [Stupid RCU Tricks: Waiting for Grace Periods From NMI Handlers](<https://paulmck.livejournal.com/65800.html>) or slides 34-on in the [Reclamation Interactions with RCU](<https://drive.google.com/file/d/1piN3sUrYJd9CwkY9jXGxPTvbGXwT-qvG/view?usp=sharing>) LSFMM+BPF 2024 presentation. 

####  Tasks Rude RCU and Tasks Trace RCU

The 2019 edition of the RCU API described the addition of Tasks RCU for use by ftrace and kprobes. These facilities use trampolines containing tracepoint code, and Tasks RCU is used to synchronize removal of a trampoline with tasks that might still be executing within it. Because ftrace and kprobes trampolines never do context switches, nor do they invoke functions that do context switches, a voluntary context switch suffices as a Tasks RCU quiescent state. This edition describes Tasks Rude RCU, which was consolidated from an open-coded implementation provided by Steve Rostedt, and also Tasks Trace RCU, which was added for use by [sleepable BPF programs](<https://lwn.net/Articles/825415/>) that might block. 

Tasks Rude RCU augments Tasks RCU by handling the idle tasks that Tasks RCU ignores, thus permitting trampolines to be installed in the idle loop. One could instead prohibit tracepoints and kprobes in the idle loop, but the increasing quantity of power-management code living there makes such a prohibition unpalatable. 

Therefore, true to its name, Tasks Rude RCU uses the `schedule_on_each_cpu()` function to force a context switch on each CPU, and thus force each CPU out of the idle loop. Those using battery-powered systems might well consider the resulting wakeups of deep-idle-state CPUs to be quite rude, hence the name. 

Like Tasks RCU, Tasks Rude RCU has no read-side markers. It has `synchronize_rcu_tasks_rude()` and `call_rcu_tasks_rude()` functions to wait for a grace period, synchronously and asynchronously, respectively. The `rcu_barrier_tasks_rude()` function waits for the invocation of all callbacks queued by previous invocations of `call_rcu_tasks_rude()`, which is needed when unloading modules such as `rcutorture` that invoke `call_rcu_tasks_rude()`. However, neither `call_rcu_tasks_rude()` nor `rcu_barrier_tasks_rude()` is used in current mainline, which is likely to lead to their removal sooner rather than later. 

Peter Zijlstra and Thomas Gleixner reworked the x86 entry/exit code, using [`noinstr`](<https://docs.kernel.org/core-api/entry.html#non-instrumentable-code-noinstr>) and inlining, so that any function that RCU is not watching cannot be traced. On any architecture where this is completed, where the CPU-hotplug architecture-specific offline code path has been addressed, and where the maintainers feel confident that it will stay completed, `synchronize_rcu_tasks_rude()` can become a no-op and `call_rcu_tasks_rude()` can invoke its callback immediately from a clean context. 

BPF programs use a combination of RCU, Tasks RCU, and Tasks Rude RCU for various purposes, including synchronizing uses of and removals of trampolines. RCU is used to protect entire BPF programs, which works well, but which prohibits BPF programs from blocking. This prohibition in turn prevents BPF programs from unconditionally loading the contents of user-space memory because of the fact that user-space accesses might result in page faults. That, in turn, might result in blocking, waiting for that user-space data to be paged back in. Therefore, BPF programs have been given conditional access to user-space memory, which either completes the access or indicates failure. 

These failure indications can be quite inconvenient, so a new special-purpose Tasks Trace RCU flavor has been created that allows limited blocking within its read-side critical sections. As such, Tasks Trace RCU can be thought of as variant of sleepable RCU (SRCU) with low-overhead read-side markers. Although, like SRCU, the implementation can tolerate arbitrary blocking, by convention Tasks Trace RCU readers are only permitted to block for long enough to handle a major page fault. 

The Tasks Trace RCU read-side markers are `rcu_read_lock_trace()` and `rcu_read_unlock_trace()`, and there is a lockdep-enabled `rcu_read_lock_trace_held()` function that indicates whether or not it is within an Tasks Trace RCU read-side critical section. (When lockdep is disabled, this function always returns the value one.) Synchronous grace-period waits are provided by `synchronize_rcu_tasks_trace()` and asynchronous grace-period waits by `call_rcu_tasks_trace()`. The `rcu_barrier_tasks_trace()` function waits for the invocation of all callbacks queued by previous invocations of `call_rcu_tasks_trace()`. It is not unusual for BPF code to need to wait for both an RCU and an Tasks Trace RCU grace period, and a current accident of implementation means that any Tasks Trace RCU grace period is also a plain RCU grace period. This could of course change at any time, so there is a `rcu_trace_implies_rcu_gp()` function (which currently unconditionally returns `true`) that specifies whether or not this happy accident is still in effect. 

Again, Tasks Trace RCU is quite specialized, so those wishing to use it should consult not only with its maintainers, but also with its current users. 

####  New SRCU read-side critical sections

SRCU read-side critical sections use `this_cpu_inc()`, which excludes interrupt and software-interrupt handlers, but is not guaranteed to exclude non-maskable interrupt (NMI) handlers. Therefore, SRCU read-side critical sections may not be used in NMI handlers, at least not in portable code. This restriction became problematic for `printk()`, which is frequently called upon to do stack backtraces from NMI handlers. This situation motivated adding `srcu_read_lock_nmisafe()` and `srcu_read_unlock_nmisafe()`. These new API members instead use `atomic_long_inc()`, which can be more expensive than `this_cpu_inc()`, but which does exclude NMI handlers. 

However, SRCU will complain if you use both the traditional and the NMI-safe API members on the same `srcu_struct` structure. In theory, it is possible to mix and match but, in practice, the rules for safely doing so are not consistent with good software-engineering practice. So if you need any of a given `srcu_struct` structure's read-side critical sections to appear in an NMI handler, use `srcu_read_lock_nmisafe()` and `srcu_read_unlock_nmisafe()` to mark all of that `srcu_struct` structure's read-side critical sections. When lockdep is enabled, the kernel will complain bitterly if you attempt to mix and match NMI-safe and non-NMI-safe SRCU readers on the same `srcu_struct` structure. 

An SRCU read-side critical section must be wholly contained within a given task. Discussions led to the belief that this restriction was too severe, resulting in the new `srcu_down_read()` and `srcu_up_read()` API members, by analogy to `down_read()` and `up_read()`. However, these APIs have not yet seen any use. If they continue to be unused, they will be removed. 

In addition, the `list_for_each_entry_srcu()` and `hlist_for_each_entry_srcu()` iterators were added, and are actually in use. 

Finally, the `cleanup_srcu_struct_quiesced()` was removed because the deadlock issue that led to its creation was resolved by adding `WQ_MEM_RECLAIM` workqueues. Therefore, any code that would previously have used `cleanup_srcu_struct_quiesced()` can now use `cleanup_srcu_struct()` instead. 

####  Read-side guards

Zijlstra introduced read-side guards for RCU and SRCU and Johannes Berg made the RCU read-side guards safe for the `sparse` static-analysis tool. These guards use the `__cleanup__` attribute to cause a read-side critical section to be exited as soon as the scope ends. This enables [the RAII (resource allocation is initialization) pattern](<https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization>) for RCU and SRCU, for example. From `fs/libfs.c`: 

> 

```
>      1 static inline struct dentry *get_stashed_dentry(struct dentry *stashed)
>      2 {
>      3   struct dentry *dentry;
>      4
>      5   guard(rcu)();
>      6   dentry = READ_ONCE(stashed);
>      7   if (!dentry)
>      8     return NULL;
>      9   if (!lockref_get_not_dead(&dentry->d_lockref))
>     10     return NULL;
>     11   return dentry;
>     12 }
>
```

**Quick Quiz 2** : What is with the extra pair of parentheses on line 5? 

Click for answer For lock-based guards, these would specify which lock to acquire. But RCU is global in nature, so does not need anything between the second pair of parentheses. 

Those willing to look more deeply under the covers will see that the `(rcu)` is the argument to the `guard()` macro and the `()` is an argument to the constructor function that enters the RCU read-side critical section. 

Line 5 creates an RCU read-side critical section that extends to the end of the enclosing scope, that is, to the end of the function body. 

An SRCU read-side guard must specify which `srcu_struct` to use, for example, as follows:
