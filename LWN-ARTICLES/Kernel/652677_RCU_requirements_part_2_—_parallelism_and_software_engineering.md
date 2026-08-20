---
title: RCU requirements part 2 — parallelism and software engineering
url: https://lwn.net/Articles/652677/
date: "August 5, 2015"
category: "Read-copy-update"
author: "August 5, 2015 This article was contributed by Paul McKenney"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

August 5, 2015

This article was contributed by Paul McKenney

This is the second of a three-article series on the requirements that have driven the read-copy-update (RCU) design toward its current form. [Part 1](<https://lwn.net/Articles/652156/>) covered the fundamental requirements that make RCU what it is. In this installment, we'll get into the issues that affect how those fundamental requirements are met; in particular, we'll look at: 

  * Parallelism facts of life
  * Quality-of-implementation requirements
  * Software-engineering requirements 

Also, of course, no RCU article would be complete without the answers to the quick quizzes at the end. 

#### Parallelism facts of life

These parallelism facts of life are by no means specific to RCU, but the RCU implementation must abide by them. They therefore bear repeating: 

  1. Any CPU or task may be delayed at any time, and any attempts to avoid these delays by disabling preemption, interrupts, or whatever are completely futile. This is most obvious in preemptible user-level environments and in virtualized environments (where a given guest OS's virtual CPUs can be preempted at any time by the underlying hypervisor), but can also happen in bare-metal environments due to ECC errors, NMIs, and other hardware events. Although a delay of more than about 20 seconds can result in warnings, the RCU implementation is obligated to use algorithms that can tolerate extremely long delays, but where "extremely long" is not long enough to allow wraparound when incrementing a 64-bit counter. 
  2. Both the compiler and the CPU can reorder memory accesses. Where it matters, RCU must use compiler directives and memory-barrier instructions to preserve ordering. 
  3. Conflicting writes to memory locations in any given cache line will result in expensive cache misses. Greater numbers of concurrent writes and more-frequent concurrent writes will result in more dramatic slowdowns. RCU is therefore obligated to use algorithms that have sufficient locality to avoid significant performance and scalability problems. 
  4. As a rough rule of thumb, only one CPU's worth of processing may be carried out under the protection of any given exclusive lock. RCU must therefore use scalable locking designs. 
  5. Counters are finite, especially on 32-bit systems. RCU's use of counters must therefore tolerate counter wrap, or be designed such that counter wrap would take way more time than a single system is likely to run. An uptime of ten years is quite possible, a runtime of a century much less so. As an example of the latter, RCU's dyntick-idle nesting counter allows 54 bits for the interrupt-nesting level (this counter is 64 bits even on a 32-bit system). Overflowing this counter requires 254 half-interrupts on a given CPU without that CPU ever going idle. If a half-interrupt happened every microsecond, it would take 570 years of runtime to overflow this counter, which is currently believed to be an acceptably long time. 
  6. Linux systems can have thousands of CPUs running a single Linux kernel in a single shared-memory environment. RCU must therefore pay close attention to high-end scalability. 

This last parallelism fact of life means that RCU must pay special attention to the preceding facts of life. The idea that Linux might scale to systems with thousands of CPUs would have been met with some skepticism in the 1990s, but these requirements would have otherwise have been unsurprising, even in the early 1990s. 

#### Quality-of-implementation requirements

These sections list quality-of-implementation requirements. Although an RCU implementation that ignores these requirements could still be used, it would likely be subject to limitations that would make it inappropriate for industrial-strength production use. Classes of quality-of-implementation requirements are as follows: 

  1. Specialization
  2. Performance and scalability
  3. Composability
  4. Corner cases 

These classes are covered in the following sections. 

#### Specialization

RCU is and always has been intended primarily for read-mostly situations, as illustrated by the following figure. This means that RCU's read-side primitives are optimized, often at the expense of its update-side primitives. 

> ![RCU Applicability](https://static.lwn.net/images/2015/rcu/RCUApplicability.png)

**Quick Quiz 11** : What about sleeping locks?   
Answer

This focus on read-mostly situations means that RCU must interoperate with other synchronization primitives. For example, the `add_gp()` and `remove_gp_synchronous()` examples discussed earlier use RCU to protect readers and locking to coordinate updaters. However, the need extends much farther, requiring that a variety of synchronization primitives be legal within RCU read-side critical sections, including spinlocks, sequence locks, atomic operations, reference counters, and memory barriers. 

It often comes as a surprise that many algorithms do not require a consistent view of data, but many can function in that mode, with network routing being the poster child. Internet routing algorithms take significant time to propagate updates, so that by the time an update arrives at a given system, that system has been sending network traffic the wrong way for a considerable length of time. Having a few threads continue to send traffic the wrong way for a few more milliseconds is clearly not a problem: In the worst case, TCP retransmissions will eventually get the data where it needs to go. In general, when tracking the state of the universe outside of the computer, some level of inconsistency must be tolerated due to speed-of-light delays if nothing else. 

Furthermore, uncertainty about external state is inherent in many cases. For example, a pair of veterinarians might use heartbeat to determine whether or not a given cat was alive. But how long should they wait after the last heartbeat to decide that the cat is in fact dead? Waiting less than 400 milliseconds makes no sense because this would mean that a relaxed cat would be considered to cycle between death and life more than 100 times per minute. Moreover, just as with human beings, a cat's heart might stop for some period of time, so the exact wait period is a judgment call. One of our pair of veterinarians might wait 30 seconds before pronouncing the cat dead, while the other might insist on waiting a full minute. The two veterinarians would then disagree on the state of the cat during the final 30 seconds of the minute following the last heartbeat. 

Interestingly enough, this same situation applies to hardware. When push comes to shove, how do we tell whether or not some external server has failed? We send messages to it periodically, and declare it failed if we don't receive a response within a given period of time. Policy decisions can usually tolerate short periods of inconsistency. The policy was decided some time ago, and is only now being put into effect, so a few milliseconds of delay is normally inconsequential. 

However, there are algorithms that absolutely must see consistent data. For example, the translation between a user-level System V semaphore ID to the corresponding in-kernel data structure is protected by RCU, but it is absolutely forbidden to update a semaphore that has just been removed. In the Linux kernel, this need for consistency is accommodated by acquiring spinlocks located in the in-kernel data structure from within the RCU read-side critical section, and this is indicated by the green box in the figure above. Many other techniques may be used, and are in fact used within the Linux kernel. 

In short, RCU is not required to maintain consistency, and other mechanisms may be used in concert with RCU when consistency is required. RCU's specialization allows it to do its job extremely well, and its ability to interoperate with other synchronization mechanisms allows the right mix of synchronization tools to be used for a given job. 

#### Performance and scalability

Energy efficiency is a critical component of performance today, and Linux-kernel RCU implementations must therefore avoid unnecessarily awakening idle CPUs. I cannot claim that this requirement was premeditated. In fact, I learned of it during a telephone conversation in which I was given "frank and open" feedback on the importance of energy efficiency in battery-powered systems and on specific energy-efficiency shortcomings of the Linux-kernel RCU implementation. In my experience, the battery-powered embedded community will consider any unnecessary wakeups to be extremely unfriendly acts. So much so that mere Linux-kernel-mailing-list posts are insufficient to vent their ire. 

Memory consumption is not particularly important in most situations, and it has become decreasingly so as memory sizes have expanded and memory costs have plummeted. However, as I learned from Matt Mackall's [bloatwatch](<http://elinux.org/Linux_Tiny-FAQ>) efforts, memory footprint is critically important on single-CPU systems with non-preemptible (`CONFIG_PREEMPT=n`) kernels, and thus [tiny RCU](<http://lkml.org/lkml/2009/1/14/449>) was born. Josh Triplett has since taken over the small-memory banner with his [Linux kernel tinification](<https://tiny.wiki.kernel.org/>) project, which resulted in [SRCU](<https://lwn.net/Articles/202847/>) becoming optional for those kernels not needing it. 

The remaining performance requirements are, for the most part, unsurprising. For example, in keeping with RCU's read-side specialization, `rcu_dereference()` should have negligible overhead (for example, suppression of a few minor compiler optimizations). Similarly, in non-preemptible environments, `rcu_read_lock()` and `rcu_read_unlock()` should have exactly zero overhead. 

In preemptible environments, in the case where the RCU read-side critical section was not preempted (as will be the case for the highest-priority realtime process), `rcu_read_lock()` and `rcu_read_unlock()` should have minimal overhead. In particular, they should not contain atomic read-modify-write operations, memory-barrier instructions, preemption disabling, interrupt disabling, or backward branches. However, in the case where the RCU read-side critical section was preempted, `rcu_read_unlock()` may acquire spinlocks and disable interrupts. This is why it is better to nest an RCU read-side critical section within a preempt-disable region than vice versa, at least in cases where that critical section is short enough to avoid unduly degrading realtime latencies. 

The `synchronize_rcu()` grace-period-wait primitive is optimized for throughput. It may therefore incur several milliseconds of latency in addition to the duration of the longest RCU read-side critical section. On the other hand, multiple concurrent invocations of `synchronize_rcu()` are required to use batching optimizations so that they can be satisfied by a single underlying grace-period-wait operation. For example, in the Linux kernel, it is not unusual for a single grace-period-wait operation to serve more than [1,000 separate invocations](<https://www.usenix.org/conference/2004-usenix-annual-technical-conference/making-rcu-safe-deep-sub-millisecond-response>) of `synchronize_rcu()`, thus amortizing the per-invocation overhead down to nearly zero. However, the grace-period optimization is also required to avoid measurable degradation of realtime scheduling and interrupt latencies. 

In some cases, the multi-millisecond `synchronize_rcu()` latencies are unacceptable. In these cases, `synchronize_rcu_expedited()` may be used instead, reducing the grace-period latency down to a few tens of microseconds on small systems, at least in cases where the RCU read-side critical sections are short. There are currently no special latency requirements for `synchronize_rcu_expedited()` on large systems, but, consistent with the empirical nature of the RCU specification, that is subject to change. However, there most definitely are scalability requirements: a storm of `synchronize_rcu_expedited()` invocations on 4096 CPUs should at least make reasonable forward progress. In return for its shorter latencies, `synchronize_rcu_expedited()` is permitted to impose modest degradation of realtime latency on non-idle online CPUs. That said, it will likely be necessary to take further steps to reduce this degradation, hopefully to roughly that of a scheduling-clock interrupt. 

There are a number of situations where even `synchronize_rcu_expedited()`'s reduced grace-period latency is unacceptable. In these situations, the asynchronous `call_rcu()` can be used in place of `synchronize_rcu()` as follows: 

> 

```
>      1 struct foo {
>      2   int a;
>      3   int b;
>      4   struct rcu_head rh;
>      5 };
>      6 
>      7 static void remove_gp_cb(struct rcu_head *rhp)
>      8 {
>      9   struct foo *p = container_of(rhp, struct foo, rh);
>     10 
>     11   kfree(p);
>     12 }
>     13 
>     14 bool remove_gp_asynchronous(void)
>     15 {
>     16   struct foo *p;
>     17 
>     18   spin_lock(&gp_lock);
>     19   p = rcu_access_pointer(gp);
>     20   if (!p) {
>     21     spin_unlock(&gp_lock);
>     22     return false;
>     23   }
>     24   rcu_assign_pointer(gp, NULL);
>     25   call_rcu(&p->rh, remove_gp_cb);
>     26   spin_unlock(&gp_lock);
>     27   return true;
>     28 }
>
```

**Quick Quiz 12** : Why does line 19 use `rcu_access_pointer()`? After all, `call_rcu()` on line 25 stores into the structure, which would interact badly with concurrent insertions. Doesn't this mean that `rcu_dereference()` is required?   
Answer

A definition of `struct foo` is finally needed, and appears on lines 1-5. The function `remove_gp_cb()` is passed to `call_rcu()` on line 25, and will be invoked after the end of a subsequent grace period. This gets the same effect as `remove_gp_synchronous()`, but without forcing the updater to wait for a grace period to elapse. The `call_rcu()` function may be used in a number of situations where neither `synchronize_rcu()` nor `synchronize_rcu_expedited()` would be legal, including within preempt-disable code, `local_bh_disable()` code, interrupt-disable code, and interrupt handlers. However, even `call_rcu()` is illegal within NMI handlers. The callback function (`remove_gp_cb()` in this case) will be executed within the softirq (software interrupt) environment within the Linux kernel (either within a real softirq handler or under the protection of `local_bh_disable()`). In both the Linux kernel and in user space, it is bad practice to write an RCU callback function that takes too long. Long-running operations should be relegated to separate threads or (in the Linux kernel) workqueues. 

However, all that `remove_gp_cb()` is doing is invoking `kfree()` on the data element. This is a common idiom, and is supported by `kfree_rcu()`, which allows "fire and forget" operation as shown below: 

> 

```
>      1 struct foo {
>      2   int a;
>      3   int b;
>      4   struct rcu_head rh;
>      5 };
>      6 
>      7 bool remove_gp_faf(void)
>      8 {
>      9   struct foo *p;
>     10 
>     11   spin_lock(&gp_lock);
>     12   p = rcu_dereference(gp);
>     13   if (!p) {
>     14     spin_unlock(&gp_lock);
>     15     return false;
>     16   }
>     17   rcu_assign_pointer(gp, NULL);
>     18   kfree_rcu(p, rh);
>     19   spin_unlock(&gp_lock);
>     20   return true;
>     21 }
>
```

**Quick Quiz 13** : Earlier it was claimed that `call_rcu()` and `kfree_rcu()` allowed updaters to avoid being blocked by readers. But how can that be correct, given that the invocation of the callback and the freeing of the memory (respectively) must still wait for a grace period to elapse?   
Answer

Note that `remove_gp_faf()` simply invokes `kfree_rcu()` and proceeds, without any need to pay any further attention to the subsequent grace period and `kfree()`. It is permissible to invoke `kfree_rcu()` from the same environments as for `call_rcu()`. Interestingly enough, DYNIX/ptx had the equivalents of `call_rcu()` and `kfree_rcu()`, but not `synchronize_rcu()`. This was due to the fact that RCU was not heavily used within DYNIX/ptx, so the very few places that needed something like `synchronize_rcu()` simply open-coded it. 

But what if the updater must wait for the completion of code to be executed after the end of the grace period, but has other tasks that can be carried out in the meantime? The polling-style `get_state_synchronize_rcu()` and `cond_synchronize_rcu()` functions may be used for this purpose, as shown below: 

> 

```
>      1 bool remove_gp_poll(void)
>      2 {
>      3   struct foo *p;
>      4   unsigned long s;
>      5 
>      6   spin_lock(&gp_lock);
>      7   p = rcu_access_pointer(gp);
>      8   if (!p) {
>      9     spin_unlock(&gp_lock);
>     10     return false;
>     11   }
>     12   rcu_assign_pointer(gp, NULL);
>     13   spin_unlock(&gp_lock);
>     14   s = get_state_synchronize_rcu();
>     15   do_something_while_waiting();
>     16   cond_synchronize_rcu(s);
>     17   kfree(p);
>     18   return true;
>     19 }
>
```

On line 14, `get_state_synchronize_rcu()` obtains a "cookie" from RCU, then line 15 carries out other tasks, and finally, line 16 returns immediately if a grace period has elapsed in the meantime, but otherwise waits as required. The need for `get_state_synchronize_rcu` and `cond_synchronize_rcu()` has appeared quite recently, so it is too early to tell whether they will stand the test of time. 

RCU thus provides a range of tools to allow updaters to strike the required tradeoff between latency, flexibility and CPU overhead. 

#### Composability

Composability has received much attention in recent years, perhaps in part due to the collision of multicore hardware with object-oriented techniques designed in single-threaded environments for single-threaded use. And, in theory, RCU read-side critical sections may be composed, and in fact may be nested arbitrarily deeply. In practice, as with all real-world implementations of composable constructs, there are limitations. 

Implementations of RCU for which `rcu_read_lock()` and `rcu_read_unlock()` generate no code, such as Linux-kernel RCU when `CONFIG_PREEMPT=n`, can be nested arbitrarily deeply. After all, there is no overhead. Except that if all these instances of `rcu_read_lock()` and `rcu_read_unlock()` are visible to the compiler, compilation will eventually fail due to exhausting memory, mass storage, or user patience, whichever comes first. If the nesting is not visible to the compiler, as is the case with mutually recursive functions each in its own translation unit, stack overflow will result. If the nesting takes the form of loops, either the control variable will overflow or (in the Linux kernel) you will get an RCU CPU stall warning. Nevertheless, this class of RCU implementations is one of the most composable constructs in existence. 

RCU implementations that explicitly track nesting depth are limited by the nesting-depth counter. For example, the Linux kernel's preemptible RCU limits nesting to `INT_MAX`. This should suffice for almost all practical purposes. That said, a consecutive pair of RCU read-side critical sections between which there is an operation that waits for a grace period cannot be enclosed in another RCU read-side critical section. This is because it is not legal to wait for a grace period within an RCU read-side critical section: to do so would result either in deadlock or in RCU implicitly splitting the enclosing RCU read-side critical section, neither of which is conducive to a long-lived and prosperous kernel. 

In short, although RCU read-side critical sections are highly composable, care is required in some situations, just as is the case for any other composable synchronization mechanism. 

#### Corner cases

A given RCU workload might have an endless and intense stream of RCU read-side critical sections, perhaps even so intense that there was never a point in time during which there was not at least one RCU read-side critical section in flight. RCU cannot allow this situation to block grace periods: as long as all the RCU read-side critical sections are finite, grace periods must also be finite. 

That said, preemptible RCU implementations could potentially result in RCU read-side critical sections being preempted for long durations, which has the effect of creating a long-duration RCU read-side critical section. This situation can arise only in heavily loaded systems, but systems using realtime priorities are of course more vulnerable. Therefore, RCU priority boosting is provided to help deal with this case. That said, the exact requirements on RCU priority boosting will likely evolve as more experience accumulates. 

Other workloads might have very high update rates. Although one can argue that such workloads should instead use something other than RCU, the fact remains that RCU must handle such workloads gracefully. This requirement is another factor driving batching of grace periods, but it is also the driving force behind the checks for large numbers of queued RCU callbacks in the `call_rcu()` code path. Finally, high update rates should not delay RCU read-side critical sections, although some read-side delays can occur when using `synchronize_rcu_expedited()`, courtesy of this function's use of `try_stop_cpus()`. (In the future, `synchronize_rcu_expedited()` will be converted to use lighter-weight inter-processor interrupts (IPIs), but this will still disturb readers, though to a much smaller degree.) 

Although all three of these corner cases were understood in the early 1990s, a simple user-level test consisting of `close(open(path))` in a tight loop in the early 2000s suddenly provided a much deeper appreciation of the high-update-rate corner case. This test also motivated addition of some RCU code to react to high update rates, for example, if a given CPU finds itself with more than 10,000 RCU callbacks queued, it will cause RCU to take evasive action by more aggressively starting grace periods and more aggressively forcing completion of grace-period processing. This evasive action causes the grace period to complete more quickly, but at the cost of restricting RCU's batching optimizations, thus increasing the CPU overhead incurred by that grace period. 

####  Software-engineering requirements

Between Murphy's Law and "to err is human", it is necessary to guard against mishaps and misuse: 

  1. It is all too easy to forget to use `rcu_read_lock()` everywhere that it is needed, so kernels built with `CONFIG_PROVE_RCU=y` will complain if `rcu_dereference()` is used outside of an RCU read-side critical section. Update-side code can use `rcu_dereference_protected()`, which takes a [lockdep expression](<https://lwn.net/Articles/371986/>) to indicate what is providing the protection. If the indicated protection is not provided, a lockdep "splat" (a warning and traceback) is emitted. 

Code shared between readers and updaters can use `rcu_dereference_check()`, which also takes a lockdep expression, and emits a lockdep splat if neither `rcu_read_lock()` nor the indicated protection is in place. In addition, `rcu_dereference_raw()` is used in those (hopefully rare) cases where the required protection cannot be easily described. Finally, `rcu_read_lock_held()` is provided to allow a function to verify that it has been invoked within an RCU read-side critical section. I was made aware of this set of requirements shortly after Thomas Gleixner audited a number of RCU uses. 
  2. A given function might wish to check for RCU-related preconditions upon entry, before using any other RCU API. The `rcu_lockdep_assert()` does this job, asserting the expression in kernels having lockdep enabled and doing nothing otherwise. 
  3. It is also easy to forget to use `rcu_assign_pointer()` and `rcu_dereference()`, perhaps (incorrectly) substituting a simple assignment. To catch this sort of error, a given RCU-protected pointer may be tagged with `__rcu`, after which running sparse with `CONFIG_SPARSE_RCU_POINTER=y` will complain about simple-assignment accesses to that pointer. Arnd Bergmann made me aware of this requirement, and also supplied the needed [patch series](<http://lwn.net/Articles/376011/>). 
  4. Kernels built with `CONFIG_DEBUG_OBJECTS_RCU_HEAD=y` will splat if a data element is passed to `call_rcu()` twice in a row, without a grace period in between. (This error is similar to a double free.) The corresponding `rcu_head` structures that are dynamically allocated are automatically tracked, but `rcu_head` structures allocated on the stack must be initialized with `init_rcu_head_on_stack()` and cleaned up with `destroy_rcu_head_on_stack()`. Similarly, statically allocated non-stack `rcu_head` structures must be initialized with `init_rcu_head()` and cleaned up with `destroy_rcu_head()`. Mathieu Desnoyers made me aware of this requirement, and also supplied the needed [patch](<https://lkml.org/lkml/2010/3/18/417>). 
  5. An infinite loop in an RCU read-side critical section will eventually trigger an RCU CPU stall-warning splat. However, RCU is not obligated to produce this splat unless there is a grace period waiting on that particular RCU read-side critical section. This requirement made itself known in the early 1990s, pretty much the first time that it was necessary to debug a CPU stall. 
  6. Although it would be very good to detect pointers leaking out of RCU read-side critical sections, there is currently no good way of doing this. One complication is the need to distinguish between pointers leaking and pointers that have been handed off from RCU to some other synchronization mechanism, for example, reference counting. 
  7. In kernels built with `CONFIG_RCU_TRACE=y`, RCU-related information is provided via both debugfs and event tracing. 
  8. Open-coded use of `rcu_assign_pointer()` and `rcu_dereference()` to create typical linked data structures can be surprisingly error-prone. Therefore, RCU-protected [linked lists](<http://lwn.net/Articles/609973/#RCU List APIs>) and, more recently, RCU-protected [hash tables](<https://lwn.net/Articles/612100/>) are available. Many other special-purpose RCU-protected data structures are available in the Linux kernel and the user-space RCU library. 
  9. Some linked structures are created at compile time, but still require `__rcu` checking. The `RCU_POINTER_INITIALIZER()` macro serves this purpose. 
  10. It is not necessary to use `rcu_assign_pointer()` when creating linked structures that are to be published via a single external pointer. The `RCU_INIT_POINTER()` macro is provided for this task and also for assigning `NULL` pointers at runtime. 

This is not a hard-and-fast list: RCU's diagnostic capabilities will continue to be guided by the number and type of usage bugs found in real-world RCU usage. 

The final installment in this series will cover issues specific to the Linux kernel and the future of RCU. 

#### Answers to the quick quizzes

**Quick Quiz 11** : What about sleeping locks? 

**Answer** : These are forbidden within Linux-kernel RCU read-side critical sections because it is not legal to place a quiescent state (in this case, voluntary context switch) within an RCU read-side critical section. However, sleeping locks may be used within user-space RCU read-side critical sections, and also within Linux-kernel sleepable RCU (SRCU) read-side critical sections. In addition, the -rt patchset turns spinlocks into a sleeping lock so that the corresponding critical sections can be preempted, which also means that these sleeplockified spinlocks (but not other sleeping locks!) may be acquired within -rt-Linux-kernel RCU read-side critical sections. 

Note that it _is_ legal for a normal RCU read-side critical section to conditionally acquire a sleeping lock (as in `mutex_trylock()`), but only as long as it does not loop indefinitely attempting to conditionally acquire that sleeping lock. The key point is that things like `mutex_trylock()` either return with the mutex held, or return an error indication if the mutex was not immediately available. Either way, `mutex_trylock()` returns immediately without sleeping. 

**Back to Quick Quiz 11**.

**Quick Quiz 12** : Why does line 19 use `rcu_access_pointer()`? After all, `call_rcu()` on line 25 stores into the structure, which would interact badly with concurrent insertions. Doesn't this mean that `rcu_dereference()` is required? 

**Answer** : Presumably the `->gp_lock` acquired on line 18 excludes any changes, including any insertions that `rcu_dereference()` would protect against. Therefore, any insertions will be delayed until after `->gp_lock` is released on line 25, which in turn means that `rcu_access_pointer()` suffices. 

**Back to Quick Quiz 12**.

**Quick Quiz 13** : Earlier it was claimed that `call_rcu()` and `kfree_rcu()` allowed updaters to avoid being blocked by readers. But how can that be correct, given that the invocation of the callback and the freeing of the memory (respectively) must still wait for a grace period to elapse? 

**Answer** : We could define things this way, but keep in mind that this sort of definition would say that updates in garbage-collected languages cannot complete until the next time the garbage collector runs, which does not seem at all reasonable. The key point is that in most cases, an updater using either `call_rcu()` or `kfree_rcu()` can proceed to the next update as soon as it has invoked `call_rcu()` or `kfree_rcu()`, without having to wait for a subsequent grace period. 

**Back to Quick Quiz 13**.
