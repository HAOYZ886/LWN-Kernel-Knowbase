---
title: "The RCU API, 2019 edition"
url: https://lwn.net/Articles/777036/
date: "January 23, 2019"
category: "Read-copy-update"
author: "January 23, 2019 This article was contributed by Paul McKenney"
---

January 23, 2019

This article was contributed by Paul McKenney

Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October 2002. RCU is most frequently described as a replacement for reader-writer locking, but has also been used in a number of other ways. RCU is notable in that readers do not directly synchronize with updaters, which makes RCU read paths extremely fast; that also permits RCU readers to accomplish useful work even when running concurrently with updaters. Although the basic idea behind RCU has not changed in decades following its introduction into DYNIX/ptx, the API has evolved significantly over the five years since the [2014 edition of the RCU API](<https://lwn.net/Articles/609904/>), to say nothing of the nine years since the [2010 edition of the RCU API](<https://lwn.net/Articles/418853/>). 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

The most recent five years of this evolution is described in the following sections: 

  * Summary of RCU API changes
  * RCU has a family of wait-to-finish and data-access APIs
  * RCU has list-based publish-subscribe and version-maintenance APIs
  * How did those 2014 predictions turn out?
  * What next for the RCU API?

These sections are followed by the answers to the quick quizzes. There is also a sidebar article on [ Kernel configuration parameters for RCU](<https://lwn.net/Articles/777214>). 

#### Summary of RCU API changes

The following sections summarize some of the most visible changes to RCU since the 2014 article. 

#### RCU flavor consolidation

The big change is the consolidation of the RCU bh, RCU preempt, and RCU sched flavors, which was carried out in response to an exploitable bug resulting from confusion between two flavors. The bug naturally led Linus Torvalds to call for a long-term solution, which ended up being this consolidation. 

Before this consolidation, these flavors could be accessed using different APIs, for example, in `CONFIG_PREEMPT=y` kernels, waiting for a grace period is done via `synchronize_rcu_bh()` for RCU bh, via `synchronize_rcu()` for RCU preempt, and via `synchronize_sched()` for RCU sched. Similarly, in `CONFIG_PREEMPT=n` kernels, `synchronize_rcu_bh()` is again used for RCU bh, but either `synchronize_rcu()` or `synchronize_sched()` may be used for RCU sched, given that RCU preempt does not exist in `CONFIG_PREEMPT=n` kernels. 

After consolidation (Linux 4.20 or later), a given running kernel has but one flavor of RCU, namely RCU preempt in `CONFIG_PREEMPT=y` kernels and RCU sched in `CONFIG_PREEMPT=n` kernels. This works because RCU preempt now waits for the completion of bh-disable, irq-disable, and preempt-disable regions of code in addition to the traditional `rcu_read_lock()`-style RCU read-side critical sections. Therefore, the vanilla RCU update-side primitives may be used in place of their RCU bh and RCU sched counterparts. Specifically: 

  * `synchronize_rcu()` may be used in place of `synchronize_rcu_bh()` and `synchronize_sched()`, as well as all current uses of `synchronize_rcu_mult()`. 
  * `synchronize_rcu_expedited()` may be used in place of `synchronize_rcu_bh_expedited()` and `synchronize_sched_expedited()`. 
  * `call_rcu()` may be used in place of `call_rcu_bh()` and `call_rcu_sched()`. 
  * `rcu_barrier()` may be used in place of `rcu_barrier_bh()` and `rcu_barrier_sched()`. 
  * `get_state_synchronize_rcu()` and `cond_synchronize_rcu()` may be used in place of `get_state_synchronize_sched()` and `cond_synchronize_sched()`, respectively. 

**Quick Quiz 1** : Why do `synchronize_rcu_mult()`, `get_state_synchronize_sched()`, and `cond_synchronize_sched()` have both blue and red backgrounds? Answer

The intent is to retire these RCU bh and RCU sched update-side APIs (along with `synchronize_rcu_mult()`, as indicated by the red backgrounds in those cells of [the big API table](<https://lwn.net/Articles/777165/#RCU Per-Flavor API Table>) (see the [key](<https://lwn.net/Articles/777165/#Key>)). 

**Quick Quiz 2** : Doesn't this consolidation also mean that it is no longer possible to wait only on preempt-disable regions of code in `CONFIG_PREEMPT=y` kernels? Answer

In addition, the RCU bh and RCU sched read-side APIs may now be used with the vanilla RCU update-side APIs, as shown in blue in the "RCU" column of the big API table, However, these read-side APIs are still separate, for example, `rcu_dereference()` will still give a lockdep complaint if it is used under `rcu_read_lock_bh()` instead of the matching `rcu_read_lock()`. 

**Quick Quiz 3** : Why not also remove `rcu_read_lock_bh()`, `rcu_read_unlock_bh()`, `rcu_read_lock_sched()`, and `rcu_read_unlock_sched()`? Answer

This change does simplify RCU updaters, for which developers no longer need to gingerly select an RCU update API that matches the readers. However, nothing comes for free, and in this case the price is that extra care is required for certain backports.

**Quick Quiz 4** : How about doing something more reliable by, for example, making the compiler complain about such backports? Answer   
  
**Quick Quiz 5** : Might we all be better off if the RCU flavors had remained unconsolidated? Answer

To see this, suppose that a bug was introduced in v4.17 in which a `synchronize_sched()` was omitted. Suppose further that this bug was fixed in v5.10 by adding a `synchronize_rcu()`. If this was a serious bug, it would of course need to be backported to the various -stable releases. Except that the `synchronize_rcu()` must be changed to `synchronize_sched()` for v4.19 and earlier. The current thought is that high-risk patches will be flagged by the -stable scripting so that yours truly (and hopefully also the respective maintainers) can inspect them.

This RCU flavor consolidation is the most profound change since 2014, but there have also been a number of other changes worth noting. The next section deals with a few additions to the sleepable RCU (SRCU) API. 

#### Addition of RCU tasks

**Quick Quiz 6** : Why not modify the scheduler-clock interrupt to check for user-mode execution? Answer   
  
**Quick Quiz 7** : Why not force a tasks RCU quiescent state by preempting all currently executing tasks? Answer

[RCU tasks](<https://lwn.net/Articles/607117/>) has voluntary context switch and user-space execution as its sole quiescent states, but applies only to non-idle tasks. This form of RCU is used by Ftrace and kprobes to manage the lifetime of "trampolines" containing the executable code used by these two facilities. The RCU tasks API consists of only `synchronize_rcu_tasks()` and `call_rcu_tasks()`, although many of the generic RCU APIs may be pressed into service as well. Note that this implementation exists only in `CONFIG_PREEMPT=y` kernels; otherwise `synchronize_rcu_tasks()` is just another name for `synchronize_rcu()` and `call_rcu_tasks()` is just another name for `call_rcu()`. 

It is important to note that RCU tasks grace periods can be quite long, in fact, the shortest they can be is about 100 milliseconds. On a busy system, they could range up into the tens of seconds and perhaps even minutes.

#### SRCU updates

**Quick Quiz 8** : Why is extreme caution required for `call_srcu()` and `srcu_barrier()`? Answer

Although the SRCU API has changed little since 2014, SRCU itself has been reimplemented (almost) from scratch one more time to provide much greater update-side scalability. It still handles readers in the idle loop and even from offline CPUs. One thing that has not changed is the need for extreme caution when using `call_srcu()`. 

Because SRCU is now used in tracepoint code, there are now `_notrace()` variants of `srcu_read_lock()`, `srcu_read_unlock()`, and `srcu_dereference()`. There is also a new `cleanup_srcu_struct_quiesced()` that is similar to `cleanup_srcu_struct()`, except that it prints an error to the console if it detects continued use of the `srcu_struct` in question. In contrast, `cleanup_srcu_struct()` will flush the `srcu_struct` structure's workqueues, but then blindly trust the caller beyond that point, which is easier to use, but can also result in hard-to-debug memory corruption. 

#### Abolition of RCU-protected array indexes

The RCU API has long permitted RCU-protected array indexes, but it turned out that these were used only in one place in x86-specific code. Because x86 has relatively strong total store order (TSO) [memory ordering](<https://en.wikipedia.org/wiki/Memory_ordering>), `rcu_dereference_index_check()` may be replaced with `smp_load_acquire()` on x86 with little or no performance penalty. In other words, in the only place using `rcu_access_index()` and `rcu_dereference_index_check()`, they were not helping much.

**Quick Quiz 9** : Given that dependencies are quite natural and intuitive, what excuse would compilers have for breaking them? And why isn't this bug being fixed? Answer

In addition, both `rcu_dereference()` (and friends) and `rcu_dereference_index_check()` rely on the compiler to preserve dependencies from those two invocations to any later dereferencing of the returned pointer. Neither the C nor the C++ standard make any sort of dependency-preservation guarantee, and compilers much more aggressively optimize integer expressions than pointer expressions, which means that dependencies carried by integers are more likely to be broken than those carried by pointers. The [rules](<https://www.kernel.org/doc/Documentation/RCU/rcu_dereference.txt>) for preventing the compiler from breaking dependencies carried by pointers are complex enough, and so it seemed best to remove integers from the mix. 

#### New data-access RCU API members

It is not unusual to fetch the value of an RCU-protected pointer, and then use `rcu_assign_pointer()` to update it, preferably under the protection of the relevant update-side lock. The shiny new `rcu_swap_protected()` combines this into a single call, saving a few lines of code. 

It is also not unheard of to access some data under RCU protection, but to also sometimes use some other synchronization mechanism in cases where longer-term protection is required. For example, the `__i915_gem_active_get_rcu()()` function picks up a pointer under RCU protection and, while still under RCU protection, checks to see if the corresponding operation has completed. If so, it returns a `NULL` pointer, for which no protection of any kind is required. Otherwise, it acquires a reference and returns the pointer, thus no longer needing RCU protection. The `rcu_pointer_handoff()` macro is used to document this hand-off from RCU to reference counter, and will hopefully also be useful to the hoped-for RCU pointer-leak diagnostic tools. 

#### RCU validation API changes

The `rcu_cpu_stall_reset()` function is used to suppress [RCU CPU stall warnings](<https://www.kernel.org/doc/Documentation/RCU/stallwarn.txt>) (also see this presentation [video [YouTube]](<https://www.youtube.com/watch?v=23_GOr8Sz-E>) and these [slides [PDF]](<http://www.rdrop.com/~paulmck/RCU/stallwarning.2018.01.22a.pdf>)). Such suppression is useful in diagnostic code which might itself cause an RCU CPU stall warning, or where a more specific diagnostic for the condition causing the warning has already been emitted. 

The `rcu_head_init()` and `rcu_head_after_call_rcu()` functions can be used to check whether a given `rcu_head` structure has already been passed to `call_rcu()`. The usage pattern is to invoke `rcu_head_init()` at allocation or initialization time, and then invoke `rcu_head_after_call_rcu()` thereafter, which will return true if that structure has already been passed to `call_rcu()`. If desired, `rcu_head_init()` could be invoked within the callback function to retrigger, but in that case providing any needed ordering with concurrent calls to `rcu_head_after_call_rcu()` is the caller's responsibility. 

The old `rcu_lockdep_assert()` facility has been renamed to `RCU_LOCKDEP_WARN()` for better alignment with other warnings. Note that the sense of the argument has also reversed, as would be expected when switching from an assertion to a warning. 

The new `rcu_sleep_check()` emits a lockdep complaint if invoked within any flavor of RCU read-side critical section. Of course, because SRCU is sleepable, `rcu_sleep_check()` emits no complaints if used within an SRCU read-side critical section. 

#### RCU-protected list changes

A number of new RCU list macros were added. The first set traverses lists protected by RCU-like mechanisms that do not require RCU read-side markers such as `rcu_read_lock()` and `rcu_read_unlock()`. One trivial example of such an RCU-like mechanism involves lists to which elements may be added but never removed. For this and similar cases, `list_for_each_entry_lockless()` traverses the list and `list_entry_lockless()` translates from a `list_head` pointer to a pointer to the enclosing structure. 

A couple additional traversal-resumption primitives have been added, namely `list_for_each_entry_from_rcu()` for normal lists and `hlist_for_each_entry_from_rcu()` for hash lists (hlists). A `hlist_nulls_for_each_entry_safe()` macro now allows deletion during traversal of hlist_nulls lists. Finally, a new `list_next_or_null_rcu()` allows easier stepwise traversal of normal lists by permitting the conventional check of the pointer against `NULL`. 

The name of `hlist_add_after_rcu()` was changed to `hlist_add_behind_rcu()` to assist a change in the order of arguments to this macro. In addition, a new `hlist_add_tail_rcu()` function allows easier appending to the ends of hlists. 

Finally, `list_splice_tail_init_rcu()` allows an RCU-protected list to be spliced onto the tail of another list. 

#### RCU has a family of wait-to-finish and data-access APIs

RCU is a four-member family of APIs as shown in the table, with four columns corresponding to one of the family members and the last column containing generic APIs that apply across families. If you are new to RCU, you might consider focusing on just one of the columns in the big RCU API table. For example, if you are primarily interested in understanding how RCU is most frequently used in the Linux kernel, "RCU" would be the place to start. On the other hand, if you want to understand RCU for its own sake, "SRCU" has the simplest API, though to a lesser extent than in the past. In both cases, you will need to refer to the final "Generic" column. You can always come back to the other columns later. If you are already familiar with RCU, this table can serve as a useful reference. 

As illustrated by the key, the green-colored RCU API members are those that existed back in the 1990s, a time when I was under the delusion that I knew all that there is to know about RCU. The blue-colored cells correspond to the RCU API members that are new since the 2014 RCU API documentation came out. The red-colored cells corresponding to RCU API members that have been deprecated (or removed entirely) since the 2014 RCU API documentation came out. 

The "RCU" column corresponds to the original RCU implementation, in which RCU read-side critical sections are delimited by a wide assortment of markers for a wide range of varieties of RCU readers, most famously `rcu_read_lock()` and `rcu_read_unlock()`. All of these markers may be nested. RCU-protected data is accessed using `rcu_dereference()` and `rcu_dereference_check()`, with the former used within RCU read-side critical sections and the latter used by code shared between readers and updaters. In both cases, the pointers must be C-language [lvalues](<https://en.wikipedia.org/wiki/Value_\(computer_science\)#lrvalue>). These read-side APIs are extremely lightweight, although the two data-access APIs execute a memory barrier on DEC Alpha. RCU's performance and scalability advantages stem from the lightweight nature of these read-side APIs. 

The corresponding synchronous update-side primitives, `synchronize_rcu()`, along with its synonym `synchronize_net()`, wait for any currently executing RCU read-side critical sections to complete. The length of this wait is known as a "grace period". If grace periods are too long for you, `synchronize_rcu_expedited()` speeds things up by about an order of magnitude, but at the expense of significant CPU overhead and of latency spikes on other CPUs. The asynchronous update-side primitive, `call_rcu()`, invokes a specified function with a specified argument after a subsequent grace period. For example, `call_rcu(p,f)` will result in the "RCU callback" `f(p)` being invoked after a subsequent grace period. There are situations, [such as when unloading a module that uses `call_rcu()`](<https://lwn.net/Articles/217484/>), when it is necessary to wait for all outstanding RCU callbacks to complete. The `rcu_barrier()` primitive does this job. The `kfree_rcu()` primitive serves as a shortcut for an RCU callback that does nothing but free the structure passed to it. Use of `kfree_rcu()` can both simplify code and reduce the need for `rcu_barrier()`. Finally, the `rcu_read_lock_held()` may be used in assertions and lockdep expressions to verify that RCU read-side protection is in fact being provided. This primitive is conservative, and thus can produce false negatives, particularly in kernels built with `CONFIG_PROVE_RCU=n`. 

The "RCU BH" column contains the RCU bh primitives. These are analogous to their RCU counterparts, and as noted earlier, the update-side RCU bh primitives are deprecated in favor of their vanilla RCU counterparts. This RCU API family was added to permit better handling of network-based denial-of-service attacks, but this functionality has since been incorporated into the vanilla RCU API. Unrelated more-aggressive transitioning from softirq context to the ksoftirqd kthreads has also helped immensely.

**Quick Quiz 10** : What happens if you mix and match RCU and RCU sched? Answer

In the "RCU Sched" column, in addition to `rcu_read_lock_sched()` and `rcu_read_unlock_sched()`, anything that disables preemption also acts as an RCU read-side critical section. Other than that, the RCU sched primitives are analogous to their RCU counterparts, though again the update-side RCU sched primitives are deprecated in favor of their vanilla RCU counterparts.

**Quick Quiz 11** : Can `synchronize_srcu()` be safely used within an SRCU read-side critical section? If so, why? If not, why not? Answer   
  
**Quick Quiz 12** : Why isn't there an `smp_mb__after_rcu_read_unlock()`, `smp_mb__after_rcu_bh_read_unlock()`, or `smp_mb__after_rcu_sched_read_unlock()`? Answer

The "SRCU" column displays a specialized RCU API that permits general sleeping in RCU read-side critical sections, as was described in the LWN article "[Sleepable RCU](<https://lwn.net/Articles/202847/>)". SRCU is also the only RCU flavor whose read-side primitives may be freely invoked from the idle loop and from offline CPUs. Of course, use of `synchronize_srcu()` in an SRCU read-side critical section can result in self-deadlock, so should be avoided. SRCU differs from earlier RCU implementations in that the caller allocates an `srcu_struct` for each distinct SRCU usage, which must either be statically allocated using either `DEFINE_SRCU()` or `DEFINE_STATIC_SRCU()` on the one hand, or be initialized after dynamic allocation using `init_srcu_struct()` on the other. This approach prevents SRCU read-side critical sections from blocking unrelated `synchronize_srcu()` invocations. In addition, in this variant of RCU, `srcu_read_lock()` returns a value that must be passed into the corresponding `srcu_read_unlock()`. There is also an `smp_mb__after_srcu_read_unlock()` that, when combined with an immediately prior `srcu_read_unlock()`, provides a full memory barrier. When finished with a runtime-initialized `srcu_struct` structure, pass it to either `cleanup_srcu_struct()` or `cleanup_srcu_struct_quiesced()`, depending on how carefully you control uses of this structure, though the latter is usually preferable. Finally, there is no counterpart to RCU's `synchronize_net()` and `kfree_rcu()` primitives. 

The "RCU tasks" column corresponds to a newly added special-purpose RCU implementation intended for the removal of the trampolines used by Ftrace and kprobes to handle the executable code that they insert into the kernel. RCU tasks features voluntary context switch and user-space execution as its sole quiescent states, but applies only to non-idle tasks. The RCU tasks API consists of only `synchronize_rcu_tasks()` and `call_rcu_tasks()`, although many of the generic RCU APIs may be pressed into service as well. 

The final column contains a few additional RCU APIs that apply equally to all five flavors. 

The following primitives do initialization: 

  * `RCU_INIT_POINTER()` may be used instead of `rcu_assign_pointer()` to assign a value to an RCU-protected pointer in a few special cases where reordering from both the compiler and the CPU can be tolerated. These special cases are as follows:
    * You are assigning `NULL` to the pointer, or 
    * You have prevented RCU readers from accessing the pointer, for example, during initialization when RCU readers do not yet have a path to the pointer in question, or 
    * The pointed-to data structure whose pointer is being assigned has already been exposed to readers, and 
      * You have not made any reader-visible changes to the pointed-to data structure since then, or 
      * It is OK for readers to see the old state of the structure  An example of this third case is when removing an element from an RCU-protected linked list, in which case that element's successor has already been exposed to readers. 
  * `RCU_POINTER_INITIALIZER()` is used for compile-time initialization of RCU-protected pointers within a structure. 

The following primitives access RCU-protected data: 

  * `rcu_access_pointer()` fetches an RCU-protected pointer in cases where ordering is not required. This primitive may be used instead of one of the `rcu_dereference()` group of primitives when only the value of the RCU-protected pointer is used without being dereferenced; for example, the RCU-protected pointer might simply be compared against `NULL`. There is therefore no need to protect against concurrent updates, and there is also no need to be under the protection of `rcu_read_lock()` and friends. 
  * `rcu_dereference_protected()` primitive is used to access RCU-protected pointers from update-side code. Because the update-side code is using some other synchronization mechanism (locks, atomic operations, single updater thread, etc.), it does not need to put RCU read-side protections in place. This primitive also takes a lockdep expression, which can be used to assert that the right locks are held and that any other necessary conditions hold. 
  * `rcu_dereference_raw()` disables lockdep checking, which allows it to be used in cases where the lockdep correctness condition cannot be expressed in a reasonably simple way. For example, the RCU list macros might be protected by any combination of RCU flavors and locks, so they use `rcu_dereference_raw()`. That said, some `_bh` list macro variants have appeared, so it is possible that lockdep-enabled variants of these macros will appear in the future. However, when you use `rcu_dereference_raw()`, please include a comment saying why its use is safe and why other forms of `rcu_dereference()` cannot be used. 
  * `rcu_dereference_raw_notrace()` is similar to `rcu_dereference_raw()`, but additionally disables function tracing. 
  * `rcu_assign_pointer()` acts like an assignment statement, but enforces store-release (as in `smp_store_release()`) ordering on both compiler and CPU as needed. The caller is responsible for providing any needed synchronization among updates. 
  * `rcu_swap_protected()` is shorthand for a call to `rcu_dereference_protected()` followed by a call to `rcu_assign_pointer()`, thus updating an RCU-protected pointer and returning the previous value. Again, the caller is responsible for providing any needed synchronization among updates. 
  * `rcu_pointer_handoff()` returns the value of the specified pointer, and indicates that something other than RCU now protects that pointer. 

Finally, the following primitives do validation: 

  * `__rcu` is used to tag RCU-protected pointers, allowing `sparse` to check for misuse of such pointers. 
  * `init_rcu_head_on_stack()` initializes an on-stack `rcu_struct` structure for debug-objects use. The [debug-objects subsystem](<https://lwn.net/Articles/271614/>) checks for memory-allocation usage bugs, for example, double `kfree()`. If the kernel is built with `CONFIG_DEBUG_OBJECTS_RCU_HEAD=y`, this checking is extended to double `call_rcu()`. Although debug-objects automatically sets up its state for global variables and heap memory, explicit setup is required for on-stack variables, hence the `init_rcu_head_on_stack()`. 
  * `destroy_rcu_head_on_stack()` must be used on any on-stack variable passed to `init_rcu_head_on_stack()` before returning from the function containing that on-stack variable. 
  * `init_rcu_head()` and `destroy_rcu_head()` also initialize objects for debug-objects use and do cleanup, respectively. These are normally not needed because the first call to `call_rcu()` will implicitly set up debug-objects state for non-stack memory. However, if that `call_rcu()` occurs in the memory allocator or in some other function used by debug-objects, this implicit `call_rcu()`-time invocation can result in deadlock. Functions called by debug-objects that also use `call_rcu()` should therefore manually invoke `init_rcu_head()` during initialization in order to break such deadlocks. 
  * `rcu_cpu_stall_reset()` disables RCU CPU stall warnings for the remainder of the current grace period, or for `ULONG_MAX / 2` jiffies, whichever comes first. This is useful for diagnostics or debugging situations in which an RCU CPU stall warning is expected behavior and in which those warnings would not be helpful. 
  * `rcu_head_init()` prepares an `rcu_head` structure for later calls to `rcu_head_after_call_rcu()`. 
  * `rcu_head_after_call_rcu()` returns true if the `rcu_head` structure has been passed to `call_rcu()`. 
  * `rcu_is_watching()` checks to see if the current code may legally contain RCU read-side critical sections. Examples of places where RCU read-side critical sections are _not_ legal include the idle loop (but see `RCU_NONIDLE()` below) and offline CPUs. Note that SRCU read-side critical sections are legal anywhere, including in the idle loop and from offline CPUs. 
  * `RCU_LOCKDEP_WARN()` is used to verify that the code has the needed protection. For example: 

```
RCU_LOCKDEP_WARN(!rcu_read_lock_held(), "Need rcu_read_lock()!");
```

This is a way to enforce the toothless comments stating that the current function must be invoked within an RCU read-side critical section. But please note that the kernel must be built with `CONFIG_PROVE_RCU=y` for this enforcement to take effect. 
  * `RCU_NONIDLE()` takes a C statement as its argument. It informs RCU that this CPU is momentarily non-idle, executes the statement, then informs RCU that this CPU is once again idle. Note that event tracing uses RCU, which means that if you are doing event tracing from the idle loop, you must use the `_rcuidle` form of the tracing functions, for example, `trace_rcu_dyntick_rcuidle()`. 
  * `rcu_sleep_check()` complains if any sort of RCU read-side critical section is in effect in kernels built with `CONFIG_PROVE_RCU=y`. 

#### RCU has list-based publish-subscribe and version-maintenance APIs

Fortunately, most of RCU's list-based publish-subscribe and version-maintenance primitives shown in the [this RCU List APIs table](<https://lwn.net/Articles/777165/#RCU List APIs>) apply to all of the variants of RCU discussed above. This commonality can in some cases allow more code to be shared, which certainly reduces the API proliferation that would otherwise occur. However, it is quite likely that software-engineering considerations will eventually result in variants of these list-handling primitives that are specialized for each given flavor of RCU, as has in fact happened with `hlist_for_each_entry_rcu_bh()` and `hlist_for_each_entry_continue_rcu_bh()`. Alternatively, perhaps it will prove useful to consolidate the read-side RCU flavors. 

The APIs in the first column of the table operate on the Linux kernel's `struct list_head` lists, which are circular, doubly-linked lists. These primitives permit lists to be modified in the face of concurrent traversals by readers. The list-traversal primitives are implemented with simple instructions, so are extremely lightweight, though they also execute a memory barrier on DEC Alpha (as does `READ_ONCE()` in recent kernels). The list-update primitives that add elements to a list incur memory-barrier overhead, while those that only remove elements from a list are implemented using simple instructions. The `list_splice_init_rcu()` and `list_splice_tail_init_rcu()` primitives incur not only memory-barrier overhead, but also grace-period latency, and are therefore the only blocking primitives shown in the table. 

The APIs in the second column of the table operate on the Linux kernel's `struct hlist_head`, which is a linear doubly linked list. One advantage of `struct hlist_head` over `struct list_head` is that the former requires only a single-pointer list header, which can save significant memory in large hash tables. The `struct hlist_head` primitives in the table relate to their non-RCU counterparts in much the same way as do the `struct list_head` primitives. Their overheads are similar to that of their list counterparts in the first two categories in the table. 

The APIs in the third column of the table operate on Linux-kernel hlist-nulls lists, which are made up of `hlist_nulls_head` and `hlist_nulls_node` structures. These lists have special multi-valued `NULL` pointers, which have the low-order bit set to 1 with the upper bits available to the programmer to distinguish different lists. There are hlist-nulls interfaces for non-RCU-protected lists as well. To see why this sort of list is useful, suppose that CPU 0 is traversing such a list within an RCU read-side critical section, where the elements are allocated from a `SLAB_TYPESAFE_BY_RCU` slab cache. The elements could therefore be freed and reallocated at any time. If CPU 0 is referencing an element while CPU 1 is freeing that element, and if CPU 1 then quickly reallocates that same element and adds it to some other list, then CPU 0 will be transported to that new list along with the element. In this case, remembering the starting list would clearly be unhelpful. 

To make matters worse, suppose that CPU 0 searches a list and fails to find the element that it was looking for. Was that because the element did not exist? Or because CPU 0 got transported to some other list in the meantime? Readers traversing `SLAB_TYPESAFE_BY_RCU` lists must carefully validate each element and check for being moved to another list. One way to check for being moved to another list is for each list to have its own value for the `NULL` pointer. These checks are subtle and easy to get wrong, so please be careful! 

A major advantage of hlist-nulls lists is that updaters can free elements to `SLAB_TYPESAFE_BY_RCU` slab caches without waiting for an RCU grace period to elapse. However, readers must be extremely careful when traversing such lists. Not only must they conduct their searches within a single RCU read-side critical section, but because any element might be freed and then reallocated at any time, readers must also validate each element that they encounter during their traversal. 

The APIs in the fourth and final column of the table operate on Linux-kernel hlist-bitlocked lists, which are made up of `hlist_bl_head` and `hlist_bl_node` structures. These lists uses the low-order bit of the pointer to the first element as a lock, which allows per-bucket locks on large hash tables while still maintaining a reasonable memory footprint. 

#### List initialization

The `INIT_LIST_HEAD_RCU()` API member allows a normal list to be initialized even when there are concurrent readers. This is useful for constructing list-splicing functions. 

#### Full traversal

The macros for full list traversal must be used within an RCU read-side critical section. These macros map to a C-language `for` loop, just as their non-RCU counterparts do. 

The individual API members are as follows: 

  * `list_for_each_entry_rcu()`: Iterate over a list from the beginning. 
  * `list_for_each_entry_lockless()`: Iterate over a insertion-only list (or similar) from the beginning. 
  * `hlist_for_each_entry_rcu()`: Iterate over an hlist from the beginning. 
  * `hlist_for_each_entry_rcu_bh()`: Iterate over a bh-protected hlist from the beginning. 
  * `hlist_for_each_entry_rcu_notrace()`: Iterate over an hlist from the beginning, but disable tracing. This macro can thus be safely used within the tracing code. 
  * `hlist_nulls_for_each_entry_rcu()`: Iterate over an hlist-nulls from the beginning. 
  * `hlist_bl_for_each_entry_rcu()`: Iterate over a bitlocked hlist from the beginning. 
  * `hlist_nulls_for_each_entry_safe()`: Iterate over a bitlocked hlist from the beginning, but allowing the current element to be deleted. 

#### Resume traversal

The macros for resuming list traversal must also be used within an RCU read-side critical section. It is the caller's responsibility to make sure that the list element that traversal has started from remains in existence, and the usual way to do this is to make sure that the RCU read-side critical section covers both the original traversal and the resumption. 

The individual API members are as follows: 

  * `list_for_each_entry_continue_rcu()`: Iterate over a list starting after specified element. 
  * `list_for_each_entry_from_rcu()`: Iterate over a list from the specified element. 
  * `hlist_for_each_entry_continue_rcu()`: Iterate over an hlist starting after the specified element. 
  * `hlist_for_each_entry_continue_rcu_bh()`: Iterate over a bh-protected hlist starting after the specified element. 
  * `hlist_for_each_entry_from_rcu()`: Iterate over an hlist from the specified element. 

#### Stepwise traversal

The macros for doing stepwise traversal are used when more control is required. When repeatedly invoking these macros to step through a list, the full set of macro invocations must be enclosed in an RCU read-side critical section. If you must split the traversal into more than one RCU read-side critical section, you must also do something else to guarantee the existence of the relevant elements during the time between successive RCU read-side critical sections. 

The individual API members are as follows: 

  * `list_entry_rcu()`: Given a pointer to a `list_head` structure, return a pointer to the enclosing element. 
  * `list_entry_lockless()`: Given a pointer to a `list_head` structure, return a pointer to the enclosing element, but don't require that the caller be within an RCU read-side critical section. 
  * `list_first_or_null_rcu()`: Return a pointer to the first element on a list, or `NULL` if the list is empty. 
  * `list_next_rcu()`: Return a pointer to the next element on the list, which must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `list_next_or_null_rcu()`: Similar to `list_next_rcu()`, except that it returns `NULL` when invoked on the last element of the list. 
  * `hlist_first_rcu()`: Return a pointer to the first element on an hlist, or `NULL` if the hlist is empty. If non-`NULL`, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_next_rcu()`: Return a pointer to the next element on an hlist, or `NULL` if at the end of the hlist. If non-`NULL`, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_pprev_rcu()`: Return a pointer to the previous element's pointer to the current element. This must not be used by RCU readers because the `->pprev` pointer is poisoned at deletion time. 
  * `hlist_nulls_first_rcu()`: Return a pointer to the first element on an hlist-nulls, or `NULL` if the hlist is empty. If non-`NULL`, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_nulls_next_rcu()`: Return a pointer to the next element on an hlist-nulls, or `NULL` if at the end of the hlist. If non-`NULL`, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_bl_first_rcu()`: Return a pointer to the first element on the bitlocked hlist, applying `rcu_dereference()`. The caller must either be in an RCU read-side critical section or have locked the hlist. Note that this cannot be an RCU bh, RCU sched, or SRCU read-side critical section, only an RCU read-side critical section will do. 

#### List addition

The list-addition APIs require some form of mutual exclusion, for example, using locking or single designated updater task. 

The individual API members are as follows: 

  * `list_add_rcu()`: Add an element at the head of the list. 
  * `list_add_tail_rcu()`: Add an element at the tail of the list. 
  * `hlist_add_before_rcu()`: Add an element before the specified element in the hlist. 
  * `hlist_add_behind_rcu()`: Add an element after the specified element in the hlist. 
  * `hlist_add_head_rcu()`: Add an element to the beginning of the hlist. 
  * `hlist_add_tail_rcu()`: Add an element to the end of the hlist. 
  * `hlist_nulls_add_head_rcu()`: Add an element to the beginning of the hlist-nulls. 
  * `hlist_bl_add_head_rcu()`: Add an element to the beginning of the bitlocked hlist. 
  * `hlist_bl_set_first_rcu()`: Add an element to the beginning of the bitlocked hlist, which must initially be empty. 

#### List deletion

The list-deletion APIs also require some form of mutual exclusion, for example, using locking or single designated updater task. 

The individual API members are as follows: 

  * `list_del_rcu()`: Delete the specified element from its list. 
  * `hlist_del_rcu()`: Delete the specified element from its hlist. 
  * `hlist_del_init_rcu()`: Delete the specified element from its hlist, initializing its pointer to form an empty list. 
  * `hlist_nulls_del_rcu()`: Delete the specified element from its hlist-nulls. 
  * `hlist_nulls_del_init_rcu()`: Delete the specified element from its hlist-nulls, initializing its pointer to form an empty list. 
  * `hlist_bl_del_rcu()`: Delete the specified element from its bitlocked hlist. 
  * `hlist_bl_del_init_rcu()`: Delete the specified element from its bitlocked hlist, initializing its pointer to form an empty list. 

#### List replacement

The list-replacement APIs replace an existing element with a new version. The caller is responsible for disposing of the old (existing) element. These also require some form of mutual exclusion, for example, using locking or single designated updater task. 

The individual API members are as follows: 

  * `list_replace_rcu()`: Replace an element in a list. 
  * `hlist_replace_rcu()`: Replace an element in an hlist. 

#### List splice

**Quick Quiz 13** : Why a mutex instead of a simple spinlock? Answer

The list-splice APIs join a pair of lists into a single combined list. This also requires some form of mutual exclusion, for example, using a mutex or a single designated updater task. 

The two API members are as follows: 

  * `list_splice_init_rcu()`: Splice a pair of lists together, initializing the source list to empty. Note that this primitive waits for a grace period to elapse. You determine the RCU flavor by passing in the corresponding grace-period-wait primitive, for example, `synchronize_rcu()` for RCU and `synchronize_rcu_bh()` for RCU bh. 
  * `list_splice_tail_init_rcu()`: Similar to `list_splice_init_rcu()`, but splices to the tail of the destination list. 

#### How did those 2014 predictions turn out?

How good were those old [predictions](<https://lwn.net/Articles/609904/#What%20Next%20for%20the%20RCU%20API?>)? 

  * "It is possible that `rcu_dereference_index_check()` will be retired if it is reasonable to convert all current use of RCU-protected indexes into RCU-protected pointers. Yes, I am doubling down on this one." 

And `rcu_dereference_index_check()` is in fact gone. 
  * "It is quite possible that large systems might encounter problems with `synchronize_rcu_expedited()` scalability. I am doubling down on this one as well, and extending it to normal grace-period operations. For example, it might be necessary to parallelize grace-period operations. For another example, it might be necessary to make `synchronize_rcu_expedited()` stop interrupting dyntick-idle CPUs." 

And `synchronize_rcu_expedited()` now uses workqueues to parallelize expedited grace period initialization. 
  * "Additional diagnostics will be required, for example, detecting pointers leaking out of RCU read-side critical sections." 

Although RCU pointer leaks still go undiagnosed, `rcutorture` has been upgraded multiple times. In addition, formal verification has been applied to the Linux-kernel RCU code [several times](<https://paulmck.livejournal.com/46993.html>). Nevertheless, much more could be done in this area, in particular, no formal verifier has yet found a bug that I didn't already know about. 
  * "A `kmem_struct` counterpart to `kfree_rcu()` will likely be required." 

There has been some work in this direction, but `kfree_rcu()` still goes unaccompanied. 
  * "Inlining of `TREE_PREEMPT_RCU`'s `rcu_read_lock()` primitive." 

And finally, `rcu_read_lock()` still goes uninlined. 

Two and a half out of five, so, as predictions go, they could have been worse. However, it is also illuminating to list the unexpected changes—all 21 of them: 

  * RCU flavor consolidation, courtesy of an exploitable security bug. 
  * Tree SRCU, that is, providing update-side scalability for SRCU. 
  * Tasks RCU, which is a specialized RCU implementation designed for trampoline removal; it uses voluntary context switch and user-mode execution as its quiescent states. 
  * Complete rework of expedited RCU grace periods, providing greater update-side scalability and allowing `synchronize_rcu_expedited()` to be invoked from within CPU-hotplug notifiers. 
  * Consolidation of (almost) all calls to `cond_resched_rcu_qs()` into `cond_resched()`, along with fixing an embarrassing consequent forward-progress issue. 
  * Nested NMIs. Or at least things that look like nested NMIs from an RCU viewpoint. 
  * Removal of the need to handle interrupts that never return as well as the need to return from interrupts that never happened. 
  * Allowing `call_rcu()` to be used at very early boot. 
  * Allowing `synchronize_rcu()` to be used during the [mid-boot dead zone](<https://lwn.net/Articles/716148/>). 
  * Addition of the `rcupdate.rcu_normal` and `rcupdate.rcu_normal_after_boot` kernel boot parameters to shield real-time workloads from IPIs due to expedited grace periods, while still allowing expedited grace periods to speed up boot. 
  * Several applications of funnel locking to avoid scalability bottlenecks. 
  * A number of interesting and embarrassing shortcomings in RCU callback offloading. 
  * Further simplification of Tiny RCU, perhaps most notably removing its ability to issue RCU CPU stall warnings. 
  * Addition of RCU CPU stall warnings to expedited RCU grace periods. 
  * The use of `rcutorture` as a stress test for other parts of the kernel. 
  * Tagging groups of RCU callbacks with grace-period numbers in order to permit idle CPUs with callbacks to sleep longer. 
  * Rewriting the interface between CPU hotplug and RCU to avoid a number of race conditions and deadlocks. 
  * The `rcutorture` scripts now automatically create the required initrd if it does not already exist. 
  * The removal of quite a few Kconfig options, along with the move of most RCU-specific Kconfig options to `kernel/rcu/Kconfig` and `kernel/rcu/Kconfig.debug`. 
  * Removal of `CONFIG_NO_HZ_FULL_SYSIDLE` due to lack of in-tree usage. 
  * Removal of RCU's debugfs tracing, again due to lack of users. 

#### What next for the RCU API?

The most honest answer is still that I do not know. That said, the following seem to be a few of the more likely directions: 

  * A `kmem_struct` counterpart to `kfree_rcu()` will likely be required. Yes, I am doubling down on this one. 
  * Inlining of `TREE_PREEMPT_RCU`'s `rcu_read_lock()` primitive. Yes, I am doubling down on this one too. 
  * Additional forward-progress work, both in `rcutorture` and in RCU proper. 
  * Better handling of [vCPU preemption within RCU readers](<https://www.usenix.org/conference/atc17/technical-sessions/presentation/prasad>). 
  * Adding `rcutorture` to kselftests, that is, adding a `Makefile` to `tools/testing/selftests/rcutorture` that carries out a quick `rcutorture`-based "smoke test" of RCU. 
  * Disentangling `rcu_barrier()` from CPU hotplug operations, which could permit this function to be invoked from CPU-hotplug notifiers. 

But if the past is any guide, new use cases and workloads will place unanticipated demands on RCU. 

#### Acknowledgments

We are all indebted to a huge number of people who have used, abused, poked at, and otherwise helped to improve the RCU API, as well as to Joel Fernandes for his careful review and thoughtful comments. I am grateful to Mark Figley for his support of this effort. 

#### Answers to quick quizzes

**Quick Quiz 1** : Why do `synchronize_rcu_mult()`, `get_state_synchronize_sched()`, and `cond_synchronize_sched()` have both blue and red backgrounds? 

**Answer** : Because they were added since 2014, but are slated for removal. As with an unfortunate but fictional legionnaire, they arrived, and then they departed. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Doesn't this consolidation also mean that it is no longer possible to wait only on preempt-disable regions of code in `CONFIG_PREEMPT=y` kernels? 

**Answer** : That is quite true. For example, `synchronize_rcu()` waits on `rcu_read_lock()`-delimited RCU read-side critical sections as well as preempt-disabled code regions. One possible concern is that `rcu_read_lock()`-delimited RCU read-side critical sections might be preempted, which could greatly increase the latency of `synchronize_rcu()` over that of pre-v4.20 `synchronize_sched()`. Should this become a problem, one way to deal with it is to build with `CONFIG_BOOST_RCU=y`, which has seen significant use by the real-time Linux community. 

It is also important to note that even the disabling of interrupts is no panacea within guest OSes, given that disabling interrupts within the guest does nothing to prevent the host OS from preempting even a interrupts-disabled vCPU. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : Why not also remove `rcu_read_lock_bh()`, `rcu_read_unlock_bh()`, `rcu_read_lock_sched()`, and `rcu_read_unlock_sched()`? 

**Answer** : That might happen some time in the future, but the current thought is that one advantage of (say) `rcu_read_lock_bh()` over `local_bh_disable()` is that `rcu_read_lock_bh()` clearly indicates that RCU is involved. In addition, some developers and maintainers might be relying on the finer-grained checking provided by lockdep, requiring for example that `rcu_dereference_bh()` be within an RCU bh read-side critical section. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : How about doing something more reliable by, for example, making the compiler complain about such backports? 

**Answer** : That was considered but rejected due to excess churn, the reason being that there are a great many more calls to `synchronize_rcu()`, `synchronize_rcu_expedited()`, `call_rcu()`, and `rcu_barrier()` than to their RCU bh and RCU sched counterparts. People are also looking into use of static analysis (for example, Coccinelle) to help with this. If you know of a better way to handle this, please don't keep it a secret. 

**Back to Quick Quiz 4**.

**Quick Quiz 5** : Might we all be better off if the RCU flavors remained unconsolidated? 

**Answer** : The consolidation does have some disadvantages, but so do exploitable security holes. In addition, the consolidation has the unintended benefit of fulfilling a long-standing request by reducing the number of `rcuo` callback-offload kthreads by a factor of two in `CONFIG_PREEMPT=n` kernels and by a factor of three in `CONFIG_PREEMPT=y` kernels, and eliminating the `rcuob` callback-offload kthreads completely. Furthermore, the consolidation reduced RCU's memory footprint slightly but significantly. Finally, the consolidation greatly simplified RCU's implementation, at least in terms of lines of code. 

All in all, therefore, this consolidation should be an overall positive. 

**Back to Quick Quiz 5**.

**Quick Quiz 6** : Why not modify the scheduler-clock interrupt to check for user-mode execution? 

**Answer** : Because it already does this checking. The scheduler-clock interrupt handler invokes `rcu_check_callbacks()`, which invokes the `CONFIG_PREEMPT=y` `rcu_flavor_check_callbacks()`, which, if `user` is set, invokes `rcu_note_voluntary_context_switch()`, which will result in a tasks RCU quiescent state in kernels implementing RCU tasks. 

**Back to Quick Quiz 6**.

**Quick Quiz 7** : Why not force a tasks RCU quiescent state by preempting all currently executing tasks? 

**Answer** : Because only a **voluntary** context switch constitutes a tasks RCU quiescent state, such preemption is of no help. 

**Back to Quick Quiz 7**.

**Quick Quiz 8** : Why is extreme caution required for `call_srcu()` and `srcu_barrier()`? 

**Answer** : Because SRCU readers are allowed to block indefinitely, these two primitives might take a long time to invoke their callbacks, if they invoke them at all. So if you use either `call_srcu()` or `srcu_barrier()`, it is your responsibility to make sure that readers complete in a timely fashion, lest large numbers of `call_srcu()` calls OOM the kernel or your `srcu_barrier()` refuse to ever return. Or, alternatively, it is your responsibility to make sure that your code does not care that `call_srcu()` and `srcu_barrier()` take forever to find the end of the grace period. 

**Back to Quick Quiz 8**.

**Quick Quiz 9** : Given that dependencies are quite natural and intuitive, what excuse would compilers have for breaking them? And why isn't this bug being fixed? 

**Answer** : In practice, compilers normally cannot reasonably break dependencies carried by pointers, however, integers are another matter altogether. To see this, consider an array whose size depends on a Kconfig option. If that array ends up having only one element, the compiler **knows** that the index must be zero, and an aggressively optimizing compiler just might decide to skip the computation entirely and just use the constant zero, which would break any dependency. Of course, pointers can also be zero, but this also means that non-buggy code won't be dereferencing them, thus defining this particular cause of dependency breaking out of existence. Issues with dependency breaking are discussed at length [here [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2015/p0098r0.pdf>). 

One could argue that compilers do in fact respect dependencies via the C and C++ `memory_order_consume` facility. However, this is only ever implemented by promoting it to `memory_order_acquire`, which on weakly ordered systems can result in unnecessary memory-barrier instructions on your fast paths, which might not be what you want. The reason for the promotion to `memory_order_acquire` is that compiler writers positively despise having to trace dependencies. In fact, there is a proposal to temporarily [deprecate](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0371r0.html>) `memory_order_consume`. 

So what is to be done? One [proposal [PDF]](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0190r4.pdf>) restricts dependency chains to cases where it is difficult for the compiler to break them, and further requires that pointer variables carrying dependencies be marked. Such marking might not go down well with the Linux-kernel community, which has been carrying dependencies in unmarked variables for more than 15 years, so there is a further informal proposal asking C and C++ implementations to provide a command-line option forcing the compiler to treat any pointer variable as if it had been marked. (Why informal? Because command-line options are outside of the scope of the standard.) 

There is a prototype implementation that obtains the functionality of `memory_order_consume` without actually using `memory_order_consume`, which is briefly described [here](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0750r1.html>). However, the committee was not all that happy with this approach, preferring marking of a single pointer variable to maintaining a separate variable to carry the dependency. 

So there is some progress and some hope, but not yet a complete and agreed-upon solution. 

**Back to Quick Quiz 9**.

**Quick Quiz 10** : What happens if you mix and match RCU and RCU sched? 

**Answer** : In v4.20 and later kernels, nothing: It just works. Not so much in v4.19 and earlier kernels, as discussed below. 

In a `CONFIG_TREE_RCU` or a `CONFIG_TINY_RCU` kernel, mixing these two works "by accident" because in those kernel builds, RCU and RCU sched map to the same implementation. However, this mixture is fatal in `CONFIG_TREE_PREEMPT_RCU` builds, due to the fact that RCU's read-side critical sections can then be preempted, which would permit `synchronize_sched()` to return before the RCU read-side critical section reached its `rcu_read_unlock()` call. This could in turn result in a data structure being freed before the read-side critical section was finished with it, which could in turn greatly increase the actuarial risk experienced by your kernel. 

Even in `CONFIG_TREE_RCU` and `CONFIG_TINY_RCU` builds, such mixing and matching is of course strongly discouraged. Mixing and matching other flavors of RCU is even worse: it can result in hard-to-debug bad-pointer bugs. 

**Back to Quick Quiz 10**.

**Quick Quiz 11** : Can `synchronize_srcu()` be safely used within an SRCU read-side critical section? If so, why? If not, why not? 

**Answer** : In theory, you can use `synchronize_srcu()` with a given `srcu_struct` within an SRCU read-side critical section that uses some other `srcu_struct`. In practice, however, such use is almost certainly a bad idea, as it means that the SRCU readers take a long time to complete. Worse yet, the following could still result in deadlock: 

> 

```
>     idx = srcu_read_lock(&ssa);
>     synchronize_srcu(&ssb);
>     srcu_read_unlock(&ssa, idx);
>     
>     /* . . . */
>     
>     idx = srcu_read_lock(&ssb);
>     synchronize_srcu(&ssa);
>     srcu_read_unlock(&ssb, idx);
>
```

The reason that this code fragment can result in deadlock is that we have a cycle. The `ssa` read-side critical sections can wait on an `ssb` grace period, which waits on `ssb` read-side critical sections, which contains a `synchronize_srcu()`, which in turn waits on `ssa` read-side critical sections. 

So if you do include `synchronize_srcu()` in SRCU read-side critical sections, make sure to avoid cycles. Of course, the simplest way to avoid cycles is to avoid using `synchronize_srcu()` in SRCU read-side critical sections in the first place. 

**Back to Quick Quiz 11**.

**Quick Quiz 12** : Why isn't there an `smp_mb__after_rcu_read_unlock()`, `smp_mb__after_rcu_bh_read_unlock()`, or `smp_mb__after_rcu_sched_read_unlock()`? 

**Answer** : Because these primitives never imply any sort of barrier. In contrast, the current implementation of `srcu_read_unlock()` actually does imply a full barrier, so `smp_mb__after_srcu_read_unlock()` can be an informative no-op. 

**Back to Quick Quiz 12**.

**Quick Quiz 13** : Why a mutex instead of a simple spinlock? 

**Answer** : Because both of these primitives wait for a grace period, which is not something you should be doing while holding a spinlock. 

**Back to Quick Quiz 13**.
