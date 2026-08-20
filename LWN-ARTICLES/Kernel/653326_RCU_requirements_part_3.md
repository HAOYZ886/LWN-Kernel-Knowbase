---
title: RCU requirements part 3
url: https://lwn.net/Articles/653326/
date: "August 12, 2015"
category: "Read-copy-update"
author: "August 12, 2015 This article was contributed by Paul McKenney"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

August 12, 2015

This article was contributed by Paul McKenney

This is the third and final segment in a series on the requirements for the kernel's read-copy-update (RCU) subsystem and how they came to be. [Part 1](<https://lwn.net/Articles/652156/>) covered the fundamental requirements that make RCU what it is, while [part 2](<https://lwn.net/Articles/652677/>) got into the requirements driven by software-engineering and parallelism concerns. In this concluding segment, the discussion will become more Linux-specific. In particular, the following topics are covered: 

  * Linux kernel complications
  * Other RCU flavors
  * Possible future changes 

Needless to say, it wouldn't be an RCU article without the answers to the quick quizzes at the end — but this article only has a single quiz in it. 

#### Linux kernel complications

The Linux kernel provides an interesting environment for all kinds of software, including RCU. Some of the relevant points of interest are as follows: 

  1. Configuration. 
  2. Firmware interface. 
  3. Early boot. 
  4. Interrupts and non-maskable interrupts (NMIs). 
  5. Loadable modules. 
  6. Hotplug CPU. 
  7. Scheduler and RCU. 
  8. Tracing and RCU. 
  9. Energy efficiency. 
  10. Performance, scalability, response time, and reliability. 

This list is probably incomplete, but it does give a feel for the most notable Linux kernel complications. Each of the following sections covers one of the above topics. 

#### Configuration

RCU's goal is automatic configuration, so that almost nobody needs to worry about RCU's `Kconfig` options. And, for almost all users, RCU does in fact work well "out of the box."

However, there are specialized use cases that are handled by kernel boot parameters and `Kconfig` options. Unfortunately, the `Kconfig` system will explicitly ask users about new `Kconfig` options, which requires almost all of them be hidden behind a `CONFIG_RCU_EXPERT` `Kconfig` option. Commits doing this hiding are making their way to mainline. 

This all should be quite obvious, but the fact remains that Linus Torvalds recently had to [remind](<http://marc.info/?l=linux-kernel&m=142905739823385>) me of this requirement. 

#### Firmware interface

In many cases, the kernel obtains information about the system from the firmware, and sometimes things are lost in translation. Or the translation is accurate, but the original message is bogus. 

For example, some systems' firmware over-reports the number of CPUs, sometimes by a large factor. If RCU naively believed the firmware, as it used to do, it would create too many per-CPU kernel threads. Although the resulting system will still run correctly, the extra threads needlessly consume memory and can cause confusion when they show up in `ps` listings. 

RCU must therefore wait for a given CPU to actually come online before it can allow itself to believe that the CPU actually exists. The resulting "ghost CPUs" (which are never going to come online) cause a number of [interesting complications](<http://paulmck.livejournal.com/37494.html>). 

#### Early boot

The Linux kernel's boot sequence is an interesting process, and RCU is used early, even before `rcu_init()` is invoked. In fact, a number of RCU's primitives can be used as soon as the initial task's `task_struct` is available and the boot CPU's per-CPU variables are set up. The read-side primitives (`rcu_read_lock()`, `rcu_read_unlock()`, `rcu_dereference()`, and `rcu_access_pointer()`) will operate normally early on, as will `rcu_assign_pointer()`. 

Although `call_rcu()` may be invoked at any time during boot, callbacks are not guaranteed to be invoked until after the scheduler is fully up and running. This delay in callback invocation is due to the fact that RCU does not invoke callbacks until it is fully initialized, and this full initialization cannot occur until after the scheduler has initialized itself to the point where RCU can spawn and run its kernel threads. In theory, it would be possible to invoke callbacks earlier; however, this is not a panacea because there would be severe restrictions on what operations those callbacks could invoke. 

Perhaps surprisingly, `synchronize_rcu()`, `synchronize_rcu_bh()` (discussed below), and `synchronize_sched()` will all operate normally during very early boot, the reason being that there is only one CPU and preemption is disabled. This means that the call `synchronize_rcu()` (or friends) itself is a quiescent state and thus a grace period, so the early-boot implementation can be a no-op. 

**Quick Quiz 14** : So what happens with `synchronize_rcu()` during scheduler initialization for `CONFIG_PREEMPT=n` kernels?   
Answer

Both `synchronize_rcu_bh()` and `synchronize_sched()` continue to operate normally through the remainder of boot, courtesy of the fact that preemption is disabled across their RCU read-side critical sections and also courtesy of the fact that there is still only one CPU. However, once the scheduler starts initializing, preemption is enabled. There is still only a single CPU, but the fact that preemption is enabled means that the no-op implementation of `synchronize_rcu()` no longer works in `CONFIG_PREEMPT=y` kernels. Therefore, as soon as the scheduler starts initializing, the early-boot fastpath is disabled. This means that `synchronize_rcu()` switches to its runtime mode of operation where it posts callbacks, which in turn means that any call to `synchronize_rcu()` will block until the corresponding callback is invoked. Unfortunately, the callback cannot be invoked until RCU's runtime grace-period machinery is up and running, which cannot happen until the scheduler has initialized itself sufficiently to allow RCU's kernel threads to be spawned. Therefore, invoking `synchronize_rcu()` during scheduler initialization can result in deadlock. 

I learned of these boot-time requirements as a result of a series of system hangs. 

#### Interrupts and NMIs

The Linux kernel has interrupts, and RCU read-side critical sections are legal within interrupt handlers and within interrupt-disabled regions of code, as are invocations of `call_rcu()`. 

Some Linux kernel architectures can enter an interrupt handler from non-idle process context, and then just never leave it, instead stealthily transitioning back to process context. This trick is sometimes used to invoke system calls from inside the kernel. These "half-interrupts" mean that RCU has to be very careful about how it counts interrupt nesting levels. I learned of this requirement the hard way during a rewrite of RCU's dyntick-idle code. 

The Linux kernel has non-maskable interrupts (NMIs), and RCU read-side critical sections are legal within NMI handlers. Thankfully, RCU update-side primitives, including `call_rcu()`, are prohibited within NMI handlers. 

The name notwithstanding, some Linux kernel architectures can have nested NMIs, which RCU must handle correctly. Andy Lutomirski recently [surprised me](<https://lkml.org/lkml/2014/11/21/642>) with this requirement; he also kindly surprised me with [an algorithm](<https://lkml.org/lkml/2014/11/22/1>) that meets that requirement. 

#### Loadable modules

The Linux kernel has loadable modules, and these modules can also be unloaded. After a given module has been unloaded, any attempt to call one of its functions results in a segmentation fault. The module-unload functions must therefore cancel any delayed calls to loadable-module functions; for example, any outstanding `mod_timer()` must be dealt with via `del_timer_sync()` or similar. 

Unfortunately, there is no way to cancel an RCU callback; once you invoke `call_rcu()`, the callback function is going to eventually be invoked, unless the system goes down first. Because it is normally considered socially irresponsible to crash the system in response to a module unload request, we need some other way to deal with in-flight RCU callbacks. 

RCU therefore provides `[rcu_barrier()](<http://lwn.net/Articles/217484/>)`, which waits until all in-flight RCU callbacks have been invoked. If a module uses `call_rcu()`, its exit function should therefore prevent any future invocation of `call_rcu()`, then invoke `rcu_barrier()`. In theory, the underlying module-unload code could invoke `rcu_barrier()` unconditionally, but in practice this would incur unacceptable latencies. 

Nikita Danilov noted this requirement for an analogous filesystem-unmount situation, and Dipankar Sarma incorporated `rcu_barrier()` into RCU. The need for `rcu_barrier()` for module unloading became apparent later. 

#### Hotplug CPU

The Linux kernel supports CPU hotplug, which means that CPUs can come and go. It is of course illegal to use any RCU API member from an offline CPU. This requirement was present from day one in DYNIX/ptx, but on the other hand, the Linux kernel's CPU-hotplug implementation is "interesting."

The Linux kernel CPU-hotplug implementation has notifiers that are used to allow the various kernel subsystems (including RCU) to respond appropriately to a given CPU-hotplug operation. Most RCU operations may be invoked from CPU-hotplug notifiers, including even normal synchronous grace-period operations such as `synchronize_rcu()`. However, expedited grace-period operations such as `synchronize_rcu_expedited()` are not supported, due to the fact that current implementations block CPU-hotplug operations, which could result in deadlock. 

In addition, all-callback-wait operations such as `rcu_barrier()` are also not supported, due to the fact that there are phases of CPU-hotplug operations where the outgoing CPU's callbacks will not be invoked until after the CPU-hotplug operation ends, which could also result in deadlock. 

#### Scheduler and RCU

RCU depends on the scheduler, and the scheduler uses RCU to protect some of its data structures. This means the scheduler is forbidden from acquiring the run-queue locks and the priority-inheritance locks in the middle of an outermost RCU read-side critical section unless it also releases them before exiting that same RCU read-side critical section. This same prohibition also applies to any lock that is acquired while holding any lock to which this prohibition applies. Violating this rule results in deadlock. 

For RCU's part, the preemptible-RCU `rcu_read_unlock()` implementation must be written carefully to avoid similar deadlocks. In particular, `rcu_read_unlock()` must tolerate an interrupt where the interrupt handler invokes both `rcu_read_lock()` and `rcu_read_unlock()`. This possibility requires `rcu_read_unlock()` to use negative nesting levels to avoid destructive recursion via interrupt handler's use of RCU. 

This pair of mutual scheduler-RCU requirements came as a [complete surprise](<http://lwn.net/Articles/453002/>). 

As noted above, RCU makes use of kernel threads, and it is necessary to avoid excessive CPU-time accumulation by these threads. This requirement was no surprise, but RCU's violation of it when running context-switch-heavy workloads when built with `CONFIG_NO_HZ_FULL=y` [did come as a surprise [PDF]](<http://www.rdrop.com/users/paulmck/scalability/paper/BareMetal.2015.01.15b.pdf>). RCU has made good progress toward meeting this requirement, even for context-switch-heavy `CONFIG_NO_HZ_FULL=y` workloads, but there is room for further improvement. 

#### Tracing and RCU

It is possible to use tracing on RCU code, but tracing itself uses RCU. For this reason, `rcu_dereference_raw_notrace()` is provided for use by tracing, which avoids the destructive recursion that could otherwise ensue. This API is also used by virtualization in some architectures, where RCU readers execute in environments in which tracing cannot be used. The tracing folks both located the requirement and provided the needed fix, so this surprise requirement was relatively painless. 

#### Energy efficiency

Interrupting idle CPUs is considered socially unacceptable, especially by people with battery-powered embedded systems. RCU therefore conserves energy by detecting which CPUs are idle, including tracking CPUs that have been interrupted from idle. This is a large part of the energy-efficiency requirement, so I learned of this via an irate phone call. 

Because RCU avoids interrupting idle CPUs, it is illegal to execute an RCU read-side critical section on an idle CPU. (Kernels built with `CONFIG_PROVE_RCU=y` will splat if you try it.) The `RCU_NONIDLE()` macro and `_rcuidle` event tracing is provided to work around this restriction. In addition, `rcu_is_watching()` may be used to test whether or not it is currently legal to run RCU read-side critical sections on this CPU. I learned of the need for diagnostics on the one hand and `RCU_NONIDLE()` on the other while inspecting idle-loop code. Steven Rostedt supplied `_rcuidle` event tracing, which is used quite heavily in the idle loop. 

It is similarly socially unacceptable to interrupt an `nohz_full` CPU running in user space. RCU must therefore track `nohz_full` user-space execution. And in [`CONFIG_NO_HZ_FULL_SYSIDLE=y`](<https://lwn.net/Articles/558284/>) kernels, RCU must separately track idle CPUs on the one hand and CPUs that are either idle or executing in user space on the other. In both cases, RCU must be able to sample state at two points in time, and be able to determine whether or not some other CPU spent any time idle and/or executing in user space. 

These energy-efficiency requirements have proven quite difficult to understand and to meet; for example, there have been more than five clean-sheet rewrites of RCU's energy-efficiency code, the last of which was finally able to demonstrate [real energy savings running on real hardware [PDF]](<http://www.rdrop.com/users/paulmck/realtime/paper/AMPenergy.2013.04.19a.pdf>). As noted earlier, I learned of many of these requirements via angry phone calls; flaming me on the Linux-kernel mailing list was apparently not sufficient to fully vent their ire at RCU's energy-efficiency bugs. 

####  Performance, scalability, response time, and reliability

Expanding on the [earlier discussion](<https://lwn.net/Articles/652677/#Performance and Scalability>) (in part 2), RCU is used heavily by hot code paths in performance-critical portions of the Linux kernel's networking, security, virtualization, and scheduling subsystems. RCU must therefore use efficient implementations, especially in its read-side primitives. To that end, it would be good if preemptible RCU's implementation of `rcu_read_lock()` could be inlined, however, doing this requires resolving `#include` issues with the `task_struct` structure. 

The Linux kernel supports hardware configurations with up to 4096 CPUs, which means that RCU must be extremely scalable. Algorithms that involve frequent acquisitions of global locks or frequent atomic operations on global variables simply cannot be tolerated within the RCU implementation. RCU therefore makes heavy use of a combining tree based on the `rcu_node` structure. RCU is required to tolerate all CPUs continuously invoking any combination of RCU's runtime primitives with minimal per-operation overhead. In fact, in many cases, increasing load must _decrease_ the per-operation overhead; witness the batching optimizations for `synchronize_rcu()`, `call_rcu()`, `synchronize_rcu_expedited()`, and `rcu_barrier()`. As a general rule, RCU must cheerfully accept whatever the rest of the Linux kernel decides to throw at it. 

The Linux kernel is used for realtime workloads, especially in conjunction with the [-rt patchset](<https://rt.wiki.kernel.org/index.php/Main_Page>). The realtime-latency response requirements are such that the traditional approach of disabling preemption across RCU read-side critical sections is inappropriate. Kernels built with `CONFIG_PREEMPT=y` therefore use an RCU implementation that allows RCU read-side critical sections to be preempted. This requirement made its presence known after users made it clear that an earlier [realtime patch](<http://lwn.net/Articles/107930/>) did not meet their needs, in conjunction with some [RCU issues](<https://lkml.org/lkml/2005/3/17/199>) encountered by a very early version of the -rt patchset. 

In addition, RCU must make do with a sub-100-microsecond realtime latency budget. In fact, on smaller systems with the -rt patchset, the Linux kernel provides sub-20-microsecond realtime latencies for the whole kernel, including RCU. RCU's scalability and latency must therefore be sufficient for these sorts of configurations. To my surprise, the sub-100-microsecond realtime latency budget [ applies to even the largest systems [PDF]](<http://www.rdrop.com/users/paulmck/realtime/paper/bigrt.2013.01.31a.LCA.pdf>), up to and including systems with 4096 CPUs. This realtime requirement motivated the grace-period kernel thread, which also simplified handling of a number of race conditions. 

Finally, RCU's status as a synchronization primitive means that any RCU failure can result in arbitrary memory corruption that can be extremely difficult to debug. This means that RCU must be extremely reliable, which in practice also means that RCU must have an aggressive stress-test suite. This stress-test suite is called `rcutorture`. 

Although the need for `rcutorture` was no surprise, the current immense popularity of the Linux kernel is posing interesting--and perhaps unprecedented--validation challenges. To see this, keep in mind that there are well over one billion instances of the Linux kernel running today, given Android smartphones, Linux-powered televisions, and servers. This number can be expected to increase sharply with the advent of the celebrated Internet of Things. 

Suppose that RCU contains a race condition that manifests on average once per million years of runtime. This bug will be occurring about three times per _day_ across the installed base. RCU could simply hide behind hardware error rates, given that no one should really expect their smartphone to last for a million years. However, anyone taking too much comfort from this thought should consider the fact that in most jurisdictions, a successful multi-year test of a given mechanism, which might include a Linux kernel, suffices for a number of types of safety-critical certifications. In fact, rumor has it that the Linux kernel is already being used in production for safety-critical applications. I don't know about you, but I would feel quite bad if a bug in RCU killed someone. Which might explain my recent focus on validation and verification. 

#### Other RCU flavors

One of the more surprising things about RCU is that there are now no fewer than five _flavors_ , or API families. In addition, the primary flavor that has been the sole focus up to this point has two different implementations, non-preemptible and preemptible. The other four flavors are listed below, with requirements for each described in a separate section. 

  1. Bottom-half flavor
  2. Sched flavor
  3. Sleepable RCU
  4. Tasks RCU 

#### Bottom-half flavor

The softirq-disable (AKA "bottom-half", hence the "_bh" abbreviations) flavor of RCU, or _RCU-bh_ , was developed by Dipankar Sarma to provide a flavor of RCU that could withstand the network-based denial-of-service attacks researched by Robert Olsson. These attacks placed so much networking load on the system that some of the CPUs never exited softirq execution, which in turn prevented those CPUs from ever executing a context switch, which, in the RCU implementation of that time, prevented grace periods from ever ending. The result was an out-of-memory condition and a system hang. 

The solution was the creation of RCU-bh, which does `local_bh_disable()` across its read-side critical sections, and which uses the transition from one type of softirq processing to another as a quiescent state in addition to context switch, idle, user mode, and offline. This means that RCU-bh grace periods can complete even when some of the CPUs execute in softirq indefinitely, thus allowing algorithms based on RCU-bh to withstand network-based denial-of-service attacks. 

Because `rcu_read_lock_bh()` and `rcu_read_unlock_bh()` disable and re-enable softirq handlers, any attempt to start a softirq handler during the RCU-bh read-side critical section will be deferred. In this case, `rcu_read_unlock_bh()` will invoke softirq processing, which can take considerable time. One can, of course, argue that this softirq overhead should be associated with the code following the RCU-bh read-side critical section rather than `rcu_read_unlock_bh()`, but the fact is that most profiling tools cannot be expected to make this sort of fine distinction. For example, suppose that a three-millisecond-long RCU-bh read-side critical section executes during a time of heavy networking load. There will very likely be an attempt to invoke at least one softirq handler during that three milliseconds, but any such invocation will be delayed until the time of the `rcu_read_unlock_bh()`. This can of course make it appear at first glance as if `rcu_read_unlock_bh()` was executing very slowly. 

The [RCU-bh API](<http://lwn.net/Articles/609973/#RCU Per-Flavor API Table>) includes `rcu_read_lock_bh()`, `rcu_read_unlock_bh()`, `rcu_dereference_bh()`, `rcu_dereference_bh_check()`, `synchronize_rcu_bh()`, `synchronize_rcu_bh_expedited()`, `call_rcu_bh()`, `rcu_barrier_bh()`, and `rcu_read_lock_bh_held()`. 

#### Sched flavor

Before preemptible RCU, waiting for an RCU grace period had the side effect of also waiting for all pre-existing interrupt and NMI handlers. However, there are legitimate preemptible-RCU implementations that do not have this property, given that any point in the code outside of an RCU read-side critical section can be a quiescent state. Therefore, RCU-sched was created, which follows "classic" RCU in that an RCU-sched grace period waits for for pre-existing interrupt and NMI handlers. In kernels built with `CONFIG_PREEMPT=n`, the RCU and RCU-sched APIs have identical implementations, while kernels built with `CONFIG_PREEMPT=y` provide a separate implementation for each. 

Note well that in `CONFIG_PREEMPT=y` kernels, `rcu_read_lock_sched()` and `rcu_read_unlock_sched()` disable and re-enable preemption, respectively. This means that if there was a preemption attempt during the RCU-sched read-side critical section, `rcu_read_unlock_sched()` will enter the scheduler, with all the latency and overhead entailed. Just as with `rcu_read_unlock_bh()`, this can make it look as if `rcu_read_unlock_sched()` was executing very slowly. However, the highest-priority task won't be preempted, so that task will enjoy low-overhead `rcu_read_unlock_sched()` invocations. 

The [RCU-sched API](<http://lwn.net/Articles/609973/#RCU Per-Flavor API Table>) includes `rcu_read_lock_sched()`, `rcu_read_unlock_sched()`, `rcu_read_lock_sched_notrace()`, `rcu_read_unlock_sched_notrace()`, `rcu_dereference_sched()`, `rcu_dereference_sched_check()`, `synchronize_sched()`, `synchronize_rcu_sched_expedited()`, `call_rcu_sched()`, `rcu_barrier_sched()`, and `rcu_read_lock_sched_held()`. However, anything that disables preemption also marks an RCU-sched read-side critical section, including `preempt_disable()` and `preempt_enable()`, `local_irq_save()` and `local_irq_restore()`, and so on. 

#### Sleepable RCU

For well over a decade, someone saying "I need to block within an RCU read-side critical section" was a reliable indication that this someone did not understand RCU. After all, if you are always blocking in an RCU read-side critical section, you can probably afford to use a higher-overhead synchronization mechanism. However, that changed with the advent of the Linux kernel's notifiers, whose RCU read-side critical sections almost never sleep, but sometimes need to. This resulted in the introduction of [sleepable RCU](<https://lwn.net/Articles/202847/>), or SRCU. 

SRCU allows different domains to be defined, with each such domain defined by an instance of an `srcu_struct` structure. A pointer to this structure must be passed into each SRCU function, for example, `synchronize_srcu(&ss)`, where `ss` is the `srcu_struct` structure. The key benefit of these domains is that a slow SRCU reader in one domain does not delay an SRCU grace period in some other domain. That said, one consequence of these domains is that read-side code must pass a "cookie" from `srcu_read_lock()` to `srcu_read_unlock()`, for example, as follows: 

> 

```
>      1 int idx;
>      2 
>      3 idx = srcu_read_lock(&ss);
>      4 do_something();
>      5 srcu_read_unlock(&ss, idx);
>
```

As noted above, it is legal to block within SRCU read-side critical sections, however, with great power comes great responsibility. If you block forever in one of a given domain's SRCU read-side critical sections, then that domain's grace periods will also be blocked forever. Of course, one good way to block forever is to deadlock, which can happen if any operation in a given domain's SRCU read-side critical section can block waiting, either directly or indirectly, for that domain's grace period to elapse. For example, this results in a self-deadlock: 

> 

```
>      1 int idx;
>      2 
>      3 idx = srcu_read_lock(&ss);
>      4 do_something();
>      5 synchronize_srcu(&ss);
>      6 srcu_read_unlock(&ss, idx);
>
```

However, if line 5 acquired a mutex that was held across a `synchronize_srcu()` for domain `ss`, deadlock would still be possible. Furthermore, if line 5 acquired a mutex that was held across a `synchronize_srcu()` for some other domain `ss1`, and if an `ss1`-domain SRCU read-side critical section acquired another mutex that was held across an `ss`-domain `synchronize_srcu()`, deadlock would again be possible. Such a deadlock cycle could extend across an arbitrarily large number of different SRCU domains. Again, with great power comes great responsibility. 

Unlike the other RCU flavors, SRCU read-side critical sections can run on idle and even offline CPUs. This ability requires that `srcu_read_lock()` and `srcu_read_unlock()` contain memory barriers, which means that SRCU readers will run a bit slower than would RCU readers. It also motivates the `smp_mb__after_srcu_read_unlock()` API, which, in combination with `srcu_read_unlock()`, guarantees a full memory barrier. 

The [SRCU API](<http://lwn.net/Articles/609973/#RCU Per-Flavor API Table>) includes `srcu_read_lock()`, `srcu_read_unlock()`, `srcu_dereference()`, `srcu_dereference_check()`, `synchronize_srcu()`, `synchronize_srcu_expedited()`, `call_srcu()`, `srcu_barrier()`, and `srcu_read_lock_held()`. It also includes `DEFINE_SRCU()`, `DEFINE_STATIC_SRCU()`, and `init_srcu_struct()` APIs for defining and initializing `srcu_struct` structures. 

#### Tasks RCU

Some forms of tracing use "trampolines" to handle the binary rewriting required to install different types of probes. It would be good to be able to free old trampolines, which sounds like a job for some form of RCU. However, because it is necessary to be able to install a trace anywhere in the code, it is not possible to have read-side markers such as `rcu_read_lock()` and `rcu_read_unlock()`. In addition, it does not work to have these markers in the trampoline itself, because there would need to be instructions following `rcu_read_unlock()`. Although `synchronize_rcu()` would guarantee that execution reached the `rcu_read_unlock()`, it would not be able to guarantee that execution had completely left the trampoline. 

The solution, in the form of [Tasks RCU](<http://lwn.net/Articles/607117/>), is to have implicit read-side critical sections that are delimited by voluntary context switches, that is, calls to `schedule()`, `cond_resched_rcu_qs()`, and `synchronize_rcu_tasks()`. In addition, transitions to and from user-space execution also delimit tasks-RCU read-side critical sections. 

The tasks-RCU API is quite compact, consisting only of `call_rcu_tasks()`, `synchronize_rcu_tasks()`, and `rcu_barrier_tasks()`. 

#### Possible future changes

One of the tricks that RCU uses to attain update-side scalability is to increase grace-period latency with increasing numbers of CPUs. If this becomes a serious problem, it will be necessary to rework the grace-period state machine so as to avoid the need for the additional latency. 

Expedited grace periods scan the CPUs, so their latency and overhead increases with increasing numbers of CPUs. If this becomes a serious problem on large systems, it will be necessary to do some redesign to avoid this scalability problem. 

RCU disables CPU hotplug in a few places, perhaps most notably in the expedited grace-period and `rcu_barrier()` operations. If there is a strong reason to use expedited grace periods in CPU-hotplug notifiers, it will be necessary to avoid disabling CPU hotplug. This would introduce some complexity, so there had better be a _very_ good reason. 

The tradeoff between grace-period latency on the one hand and interruptions of other CPUs on the other hand may need to be re-examined. The desire is of course for zero grace-period latency as well as zero inter-processor interrupts undertaken during an expedited grace period operation. While this ideal is unlikely to be achievable, it is quite possible that further improvements can be made. 

The multiprocessor implementations of RCU use a combining tree that groups CPUs so as to reduce lock contention and increase cache locality. However, this combining tree does not spread its memory across NUMA nodes nor does it align the CPU groups with hardware features such as sockets or cores. Such spreading and alignment is currently believed to be unnecessary because the hot-path read-side primitives do not access the combining tree, nor does `call_rcu()` in the common case. If you believe that your architecture needs such spreading and alignment, then your architecture should also benefit from the `rcutree.rcu_fanout_leaf` boot parameter, which can be set to the number of CPUs in a socket, NUMA node, or whatever. If the number of CPUs is too large, use a fraction of the number of CPUs. If the number of CPUs is a large prime number, well, that certainly is an "interesting" architectural choice. More flexible arrangements might be considered, but only if `rcutree.rcu_fanout_leaf` has proven inadequate, and only if the inadequacy has been demonstrated by a carefully run and realistic system-level workload. 

Please note that arrangements that require RCU to remap CPU numbers will require extremely good demonstration of need and full exploration of alternatives. 

There is an embarrassingly large number of flavors of RCU, and this number has been increasing over time. Perhaps it will be possible to combine some at some future date. 

RCU's various kernel threads are reasonably recent additions. It is quite likely that adjustments will be required to more gracefully handle extreme loads. It might also be necessary to be able to relate CPU utilization by RCU's kernel threads and softirq handlers to the code that instigated this CPU utilization. For example, RCU callback overhead might be charged back to the originating `call_rcu()` instance, though probably not in production kernels. 

#### Summary

This document has presented more than two decade's worth of RCU requirements. Given that the requirements keep changing, this will not be the last word on this subject, but at least it serves to get an important subset of the requirements set forth. 

#### Acknowledgments

I am grateful to Steven Rostedt, Lai Jiangshan, Ingo Molnar, Oleg Nesterov, Borislav Petkov, Peter Zijlstra, and Andy Lutomirski for their help in rendering this article human readable, and to Michelle Rankin for her support of this effort. 

#### Answer to the quick quiz

**Quick Quiz 14** : So what happens with `synchronize_rcu()` during scheduler initialization for `CONFIG_PREEMPT=n` kernels? 

**Answer** : In `CONFIG_PREEMPT=n` kernels, `synchronize_rcu()` maps directly to `synchronize_sched()`. Therefore, `synchronize_rcu()` works normally throughout boot in `CONFIG_PREEMPT=n` kernels. However, the code must work in `CONFIG_PREEMPT=y` kernels, so it is still necessary to avoid invoking `synchronize_rcu()` during scheduler initialization. 

**Back to Quick Quiz 14**.
