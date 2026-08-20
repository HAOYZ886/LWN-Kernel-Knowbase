---
title: "The RCU API, 2014 Edition"
url: https://lwn.net/Articles/609904/
date: "September 4, 2014"
category: "Read-copy-update"
author: "September 4, 2014 This article was contributed by Paul McKenney"
---

September 4, 2014

This article was contributed by Paul McKenney

Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October 2002\. RCU is most frequently described as a replacement for reader-writer locking, but has also been used in a number of other ways. RCU is notable in that RCU readers do not directly synchronize with RCU updaters, which makes RCU read paths extremely fast, and also permits readers to accomplish useful work even when running concurrently with updaters. 

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Although the basic idea behind RCU has not changed in the decades following its introduction into DYNIX/ptx in 1993, the RCU API has evolved significantly even over the nearly four years since a [2010 article covering the RCU API](<https://lwn.net/Articles/418853/>), most recently due to software-engineering concerns. This evolution is documented by the following sections. 

  1. Summary of RCU API additions
  2. RCU has a family of wait-to-finish and data-access APIs
  3. RCU has list-based publish-subscribe and version-maintenance APIs
  4. Kernel configuration parameters
  5. How did those 2010 predictions turn out?
  6. What next for the RCU API?

These sections are followed by answers to the Quick Quizzes. 

####  Summary of RCU API additions

**Quick Quiz 1** : Why is extreme caution required for `call_srcu()` and `srcu_barrier()`?   
Answer

The largest change to the RCU API since 2010 has been to SRCU (Sleepable RCU), which was reimplemented from scratch by Peter Zijlstra, myself, and finally by Lai Jiangshan. The new implementation offers much lower-latency grace periods (which was important for KVM), and, unlike other RCU implementations, allows readers in the idle loop and even in offline CPUs. In addition, this new SRCU implementation provides the full RCU API, including the `call_srcu()` and `srcu_barrier()` functions that were omitted from the previous version. That said, these new APIs should be used with extreme caution. 

Another important addition is `kfree_rcu()`, which allows "fire and forget" freeing of RCU-protected data. Given a structure `p` with an `rcu_head` field imaginatively named `rh`, you can now free a structure pointed to by `p` as follows: 

> `kfree_rcu(p, rh);`

Before `kfree_rcu()` was available, something like this was instead required: 

> 

```
>     static void free_by_callback(struct rcu_head *rhp)
>     {
>     	struct foo *p = container_of(rhp, struct foo, rh);
>     
>     	kfree(p);
>     }
>     
>     ...
>     
>     	call_rcu(&p->rh, free_by_callback);
>
```

**Quick Quiz 2** : If `kfree_rcu()` is so popular, why isn't there a `kfree_rcu_bh()`, `kfree_rcu_sched()`, or `kfree_srcu()`? For that matter, why not a `kmem_cache_free_rcu()`?   
Answer

The use of `kfree_rcu()` therefore saves a bit of code, so that this API now has almost 200 uses in the Linux kernel. Equally important, if your loadable module uses `call_rcu()`, you will need to use `rcu_barrier()` at module-unload time, as described [here](<https://lwn.net/Articles/217484/>). If you can convert all of your module's `call_rcu()` invocations to `kfree_rcu()`, then the `rcu_barrier()` is not needed. 

A new bitlocked linked list (`hlist_bl_head` and `hlist_bl_node`) was added. The bitlocked linked list is useful when you need a lock associated with each linked list, but memory pressures prohibit associating a spinlock_t with each list. As the name suggests, a bitlocked linked list reduces the size of the lock down to a single bit that is placed in the low-order bit of a pre-existing pointer. Bitlocked linked lists required the RCU-safe accessors `hlist_bl_for_each_entry_rcu()`, `hlist_bl_first_rcu()`, `hlist_bl_add_head_rcu()`, `hlist_bl_del_rcu()`, `hlist_bl_del_init_rcu()`, and `hlist_bl_set_first_rcu()`. 

Performance issues in networking led to the addition of `RCU_INIT_POINTER()`, which can be used in place of `rcu_assign_pointer()` in a few special cases, and that omits `rcu_assign_pointer()`'s barrier and volatile cast. Ugly-code issues led to the addition of `RCU_POINTER_INITIALIZER()`, which may be used to initialize RCU-protected pointers in structures at compile time. 

The `rcu_access_index()` primitive was requested as a RCU-protected-index counterpart to `rcu_access_pointer()` for integers used as array indexes. The `rcu_access_index()` and `rcu_access_pointer()` functions may be used in cases where the index or pointer will not be dereferenced; for example, when it is just being compared. Therefore, `rcu_access_index()` and `rcu_access_pointer()` need not use `smp_read_barrier_depends()`, however, given that `smp_read_barrier_depends()` is free on most platforms, this is not much of a motivation. The motivation instead is that these primitives can be safely used outside of RCU read-side critical sections, thus avoiding the need to worry about lockdep-RCU complaints. 

The `rcu_dereference_raw_notrace()`, `rcu_is_watching()`, and `hlist_for_each_entry_rcu_notrace()` primitives were needed for special cases in the tracing code. The motivation here is that the tracing code uses RCU, but also needs to be able to trace RCU's primitives. Providing these `_notrace` variants allows the tracing implementation to more easily avoid self-recursion. 

Lower-level list APIs for stepwise traversal of RCU-protected lists were added: `list_first_or_null_rcu()`, `list_next_rcu()`, `hlist_first_rcu()`, `hlist_next_rcu()`, `hlist_pprev_rcu()`, `hlist_nulls_first_rcu()`, and `hlist_nulls_next_rcu()`. In addition, the hlist-nulls primitive `hlist_nulls_del_init_rcu()` was added as as counterpart to the hlist `hlist_del_init_rcu()` primitive. 

The `rcu_lockdep_assert()` primitive allows functions to insist that they be invoked within the specified RCU and locking contexts: Experience indicates that RCU-lockdep splats get the prompt attention required to ensure that such functions are called in the required environment. 

Finally, `RCU_NONIDLE()` may be used to protect RCU read-side critical sections in idle loops, which would otherwise be illegal. `RCU_NONIDLE()` is not used much because almost all the idle-loop uses of RCU are due to tracing, which supplies a trace functions with an `_rcuidle` suffix for idle-loop use. However, non-tracing uses of RCU within the idle loop should use `RCU_NONIDLE()`. There is some discussion of restricting the region of idle-loop code that RCU considers to be idle, and if this region becomes small enough, it might be possible to dispense with both `RCU_NONIDLE()` and the `_rcuidle` suffix. 

The next sections discuss aspects of the RCU API, highlighting recent changes. 

####  RCU has a family of wait-to-finish and data-access APIs

The most straightforward answer to "what is RCU?" is that RCU is an API used in the Linux kernel, as summarized by [the big API table](<https://lwn.net/Articles/609973/#RCU Per-Flavor API Table>) and the following discussion. Or, more precisely, RCU is a four-member family of APIs as shown in the table, with four columns that correspond to each of the family members and the last column containing generic APIs that apply across families. 

If you are new to RCU, you might consider focusing on just one of the columns in the big RCU API table. For example, if you are primarily interested in understanding how RCU is most frequently used in the Linux kernel, "RCU" would be the place to start. On the other hand, if you want to understand RCU for its own sake, "SRCU" has the simplest API. In both cases, you will need to refer to the final "Generic" column. You can always come back to the other columns later. If you are already familiar with RCU, this table can serve as a useful reference. 

The green-colored RCU API members are those that existed back in the 1990s, a time when I was under the delusion that I knew all that there is to know about RCU. The blue-colored cells correspond to the RCU API members that are new since the 2010 RCU API documentation came out. 

The "RCU" column corresponds to the original RCU implementation, in which RCU read-side critical sections are delimited by `rcu_read_lock()` and `rcu_read_unlock()`, which may be nested. RCU-protected data is accessed using `rcu_dereference()` and `rcu_dereference_check()`, with the former used within RCU read-side critical sections and the latter used by code shared between readers and updaters. In both cases, the pointers must be C-language lvalues. These read-side APIs are lightweight, although the two data-access APIs must [execute a memory barrier on DEC Alpha](<http://www.openvms.compaq.com/wizard/wiz_2637.html>). RCU's performance and scalability advantages stem from the lightweight nature of these read-side APIs. 

The corresponding synchronous update-side primitives, `synchronize_rcu()`, along with its synonym, `synchronize_net()`, wait for any currently executing RCU read-side critical sections to complete. The length of this wait is known as a "grace period". If grace periods are too long for you, `synchronize_rcu_expedited()` speeds things up by about an order of magnitude, but at the expense of significant CPU overhead and of latency spikes on all CPUs, even the CPUs that are currently idle. 

The asynchronous update-side primitive, `call_rcu()`, invokes a specified function with a specified argument after a subsequent grace period. For example, `call_rcu(p,f);` will result in the "RCU callback" `f(p)` being invoked after a subsequent grace period. There are situations, such as when unloading a module that uses `call_rcu()`, when it is necessary to wait for all outstanding RCU callbacks to complete. The `rcu_barrier()` primitive does this job. The `kfree_rcu()` primitive serves as a shortcut for an RCU callback that does nothing but free the structure passed to it. Use of `kfree_rcu()` can both simplify code and reduce the need for `rcu_barrier()`. Finally, the `rcu_read_lock_held()` may be used in assertions and lockdep expressions to verify that RCU read-side protection is in fact being provided. This primitive is conservative, and thus can produce false negatives, particularly in kernels built with `CONFIG_PROVE_RCU=n`. 

The "RCU BH" column contains the RCU-bh primitives. RCU-bh differs from RCU in that RCU-bh read-side critical sections (`rcu_read_lock_bh()` and `rcu_read_unlock_bh()`) disable bottom-half (i.e. softirq) processing. RCU-bh also features somewhat lower-latency grace periods in CONFIG_PREEMPT=n kernels due to the fact that any point in the code where bottom halves are enabled is an RCU-bh quiescent state. The overall effect is that RCU-bh trades off slightly increased read-side overhead to gain shorter and more predictable grace periods. These shorter grace periods allow the system to avoid out-of-memory (OOM) conditions in the face of network-based denial-of-service attacks. 

**Quick Quiz 3** : What happens if you mix and match RCU and RCU-Sched?   
Answer

In the "RCU Sched" column, anything that disables preemption acts as an RCU read-side critical section, however, `rcu_read_lock_sched()` and `rcu_read_unlock_sched()` are the official read-side primitives. Other than that, the RCU-sched primitives are analogous to their RCU counterparts, though, again, RCU-sched lacks a counterpart to `synchronize_net()` and `kfree_rcu()`. This RCU API family was added in the 2.6.12 kernel, which split the old `synchronize_kernel()` API into the current `synchronize_rcu()` (for RCU) and `synchronize_sched()` (for RCU Sched). There are also `call_rcu_sched()`, `synchronize_sched_expedited()`, and `rcu_barrier_sched()` primitives, which are analogous to their "RCU" counterparts. 

**Quick Quiz 4** : Can `synchronize_srcu()` be safely used within an SRCU read-side critical section? If so, why? If not, why not?   
Answer

The "SRCU" column displays a specialized RCU API that permits general sleeping in RCU read-side critical sections, as was described in the LWN article "[Sleepable RCU](<https://lwn.net/Articles/202847/>)". SRCU is also the only RCU flavor whose read-side primitives may be freely invoked from the idle loop and from offline CPUs. Of course, use of `synchronize_srcu()` in an SRCU read-side critical section can result in self-deadlock, so it should be avoided. SRCU differs from earlier RCU implementations in that the caller allocates an `srcu_struct` for each distinct SRCU usage, which must either be statically allocated using `DEFINE_SRCU()` or be initialized after dynamic allocation using `init_srcu_struct()`. 

**Quick Quiz 5** : Why isn't there an `smp_mb__after_rcu_read_unlock()`, `smp_mb__after_rcu_bh_read_unlock()`, or `smp_mb__after_rcu_sched_read_unlock()`?   
Answer

This approach prevents SRCU read-side critical sections from blocking unrelated `synchronize_srcu()` invocations. In addition, in this variant of RCU, `srcu_read_lock()` returns a value that must be passed into the corresponding `srcu_read_unlock()`. There is also an `smp_mb__after_srcu_read_unlock()` that, when combined with an immediately prior `srcu_read_unlock()`, provides a full memory barrier. Finally, as with RCU-bh and RCU-sched, there is no counterpart to RCU's `synchronize_net()` and `kfree_rcu()` primitives.

The final column contains a few additional RCU APIs that apply equally to all four flavors. 

The following primitives do initialization: 

  * `RCU_INIT_POINTER()` may be used instead of `rcu_assign_pointer()` to assign a value to an RCU-protected pointer in a few special cases where reordering from both the compiler and the CPU can be tolerated. These special cases are as follows: 

    * You are assigning `NULL` to the pointer, or 
    * You have prevented RCU readers from accessing the pointer, for example, during initialization when RCU readers do not yet have a path to the pointer in question, or 
    * The pointed-to data structure whose pointer is being assigned has already been exposed to readers, and 

      * You have not made any reader-visible changes to the pointed-to data structure since then, or 
      * It is OK for readers to see the old state of the structure.  An example of this third case is when removing an element from an RCU-protected linked list, in which case that element's successor has already been exposed to readers. 
  * `RCU_POINTER_INITIALIZER()` is used for compile-time initialization of RCU-protected pointers within a structure. 

The following primitives access RCU-protected data: 

  * `rcu_access_index()` fetches an RCU-protected index in cases where ordering is not required, for example, when the only use of the value fetched is for a comparison. 
  * `rcu_access_pointer()` fetches an RCU-protected pointer in cases where ordering is not required. This primitive may be used instead of one of the `rcu_dereference()` group of primitives when only the value of the RCU-protected pointer is used without being dereferenced; for example, the RCU-protected pointer might simply be compared against `NULL`. There is, therefore, no need to protect against concurrent updates, and there is also no need to be under the protection of `rcu_read_lock()` and friends. 
  * `rcu_dereference_index_check()` fetches an RCU-protected index, and takes a lockdep expression identifying the locks and types of RCU that protect the access. Unfortunately, this primitive also disables sparse-based checking, so it is possible that this primitive will be deprecated in the future; any remaining uses should shift from using indexes to the equivalent pointers. (So if you know of an RCU-protected index that cannot be easily converted to an RCU-protected pointer, this would be a really good time to speak up.) 
  * `rcu_dereference_protected()` is used to access RCU-protected pointers from update-side code. Because the update-side code is using some other synchronization mechanism (locks, atomic operations, single updater thread, etc.), it does not need to put RCU read-side protections in place. This primitive also takes a lockdep expression that can be used to assert that the right locks are held and that any other necessary conditions hold. 
  * `rcu_dereference_raw()` disables lockdep checking, which allows it to be used in cases where the lockdep correctness condition cannot be expressed in a reasonably simple way. For example, the RCU list macros might be protected by any combination of RCU flavors and locks, so they use `rcu_dereference_raw()`. That said, some `_bh` list-macro variants have appeared, so it is possible that lockdep-enabled variants of these macros will appear in the future. However, where you use `rcu_dereference_raw()`, please include a comment saying why its use is safe and why other forms of `rcu_dereference()` cannot be used. 
  * `rcu_dereference_raw_notrace()` is similar to `rcu_dereference_raw()`, but additionally disables function tracing. 

The following primitive updates RCU-protected data: 

  * `rcu_assign_pointer()` acts like an assignment statement, but does debug checking and enforces ordering on both compiler and CPU as needed. 

Finally, the following primitives do validation: 

  * `__rcu` is used to tag RCU-protected pointers, allowing `sparse` to check for misuse of such pointers. 
  * `init_rcu_head_on_stack()` initializes an on-stack `rcu_struct` structure for debug-objects use. The debug-objects subsystem checks for memory-allocation usage bugs, for example, double-`kfree()`. If the kernel is built with `CONFIG_DEBUG_OBJECTS_RCU_HEAD=y`, this checking is extended to double `call_rcu()`. Although debug-objects automatically sets up its state for global variables and heap memory, explicit setup is required for on-stack variables, hence the `init_rcu_head_on_stack()`. 
  * `destroy_rcu_head_on_stack()` must be used on any on-stack variable passed to `init_rcu_head_on_stack()` before returning from the function containing that on-stack variable. 
  * `init_rcu_head()` and `destroy_rcu_head()` also initialize objects for debug-objects use. These are normally not needed because the first call to `call_rcu()` will implicitly set up debug-objects state for non-stack memory. However, if that `call_rcu()` occurs in the memory allocator or in some other function used by debug-objects, this implicit `call_rcu()`-time invocation can result in deadlock. Functions called by debug-objects that also use `call_rcu()` should therefore manually invoke `debug_init_rcu_head()` during initialization in order to break such deadlocks. 
  * `rcu_is_watching()` checks to see if the current code may legally contain RCU read-side critical sections. Examples of places where RCU read-side critical sections are _not_ legal include the idle loop (but see `RCU_NONIDLE()` below) and offline CPUs. Note that SRCU read-side critical sections are legal anywhere, including in the idle loop and from offline CPUs. 
  * `rcu_lockdep_assert()` is used to verify that the code has the needed protection. For example: 

```
rcu_lockdep_assert(rcu_read_lock_held(), "Need rcu_read_lock()!");
```

is a way to enforce the toothless comments stating that the current function must be invoked within an RCU read-side critical section. But please note that the kernel must be built with `CONFIG_PROVE_RCU=y` for this enforcement to take effect. 
  * `RCU_NONIDLE()` takes a C statement as its argument. It informs RCU that this CPU is momentarily non-idle, executes the statement, then informs RCU that this CPU is once again idle. Note that event tracing uses RCU, which means that if you are doing event tracing from the idle loop, you must use the `_rcuidle` form of the tracing functions, for example: `trace_rcu_dyntick_rcuidle()`. 

The Linux kernel currently has a surprising number of RCU APIs and implementations. There is some hope of reducing this number, but careful inspection and analysis will be required before removing either an implementation or any API members, just as would be required before removing one of the many locking APIs in the kernel. Besides which, recent trends have been very much in the opposite direction. The next section describes RCU's list-based APIs, which have seen some growth since 2010\. 

####  RCU has list-based publish-subscribe and version-maintenance APIs

Although in theory `rcu_dereference()` and `rcu_assign_pointer()` are sufficient to implement pretty much any data structure, in practice this approach would be time-consuming and error-prone. RCU therefore provides specialized list-based publish-subscribe and version-maintenance APIs. Fortunately, most of RCU's list-based publish-subscribe and version-maintenance primitives shown in [this table](<https://lwn.net/Articles/609973/#RCU List APIs>) apply to all of the variants of RCU discussed above. This commonality can, in some cases, allow more code to be shared, which certainly reduces the API proliferation that would otherwise occur. However, it is quite likely that software-engineering considerations will eventually result in variants of these list-handling primitives that are specialized for each given flavor of RCU, as has, in fact, happened with `hlist_for_each_entry_rcu_bh()` and `hlist_for_each_entry_continue_rcu_bh()`. 

The APIs in the first column of the table operate on the Linux kernel's `struct list_head` lists, which are circular, doubly-linked lists. These primitives permit lists to be modified in the face of concurrent traversals by readers. The list-traversal primitives are implemented with simple instructions, so are extremely lightweight, although they also execute a memory barrier on DEC Alpha. The list-update primitives that add elements to a list incur memory-barrier overhead, while those that only remove elements from a list are implemented using simple instructions. The `list_splice_init_rcu()` primitive incurs not only memory-barrier overhead, but also grace-period latency, and is therefore the only blocking primitive shown in the table. 

The APIs in the second column of the table operate on the Linux kernel's `struct hlist_head`, which is a linear doubly linked list. One advantage of `struct hlist_head` over `struct list_head` is that the former requires only a single-pointer list header, which can save significant memory in large hash tables. The `struct hlist_head` primitives in the table relate to their non-RCU counterparts in much the same way as do the `struct list_head` primitives. Their overheads are similar to that of their list counterparts in the first two categories in the table. 

**Quick Quiz 6** : Why would anyone need to distinguish lists based on their NULL pointers? Why not just remember which list you started searching?   
Answer

The APIs in the third column of the table operate on Linux-kernel hlist-nulls lists, which are made up of `hlist_nulls_head` and `hlist_nulls_node` structures. These lists have special multi-valued `NULL` pointers, which have the low-order bit set to 1 with the upper bits available to the programmer to distinguish different lists. There are hlist-nulls interfaces for non-RCU-protected lists as well. 

A major advantage of hlist-nulls lists is that updaters can free elements to `SLAB_DESTROY_BY_RCU` slab caches without waiting for an RCU grace period to elapse. However, readers must be extremely careful when traversing such lists: Not only must they conduct their searches within a single RCU read-side critical section, but, because any element might be freed and then reallocated at any time, readers must also validate each element that they encounter during their traversal. 

**Quick Quiz 7** : Why is there no `hlist_nulls_add_tail_rcu()`?   
Answer

The APIs in the fourth and final column of the table operate on Linux-kernel hlist-bitlocked lists, which are made up of `hlist_bl_head` and `hlist_bl_node` structures. These lists use the low-order bit of the pointer to the first element as a lock, which allows per-bucket locks on large hash tables while still maintaining a reasonable memory footprint. 

#####  List Initialization

The new `INIT_LIST_HEAD_RCU()` API member allows a normal list to be initialized even when there are concurrent readers. This is useful for constructing list-splicing functions. 

#####  Full Traversal

The macros for full list traversal must be used within an RCU read-side critical section. These macros map to a C-language `for` loop, just as their non-RCU counterparts do. 

The individual API members are as follows: 

  * `list_for_each_entry_rcu()`: Iterate over an RCU-protected list from the beginning. 
  * `hlist_for_each_entry_rcu()`: Iterate over an RCU-protected hlist from the beginning. 
  * `hlist_for_each_entry_rcu_bh()`: Iterate over an RCU-bh-protected hlist from the beginning. 
  * `hlist_for_each_entry_rcu_notrace()`: Iterate over an RCU-protected hlist from the beginning, but disable tracing. This macro can thus be safely used within the tracing code. 
  * `hlist_nulls_for_each_entry_rcu()`: Iterate over an RCU-protected hlist-nulls from the beginning. 
  * `hlist_bl_for_each_entry_rcu()`: Iterate over an RCU-protected bitlocked hlist from the beginning. 

#####  Resume Traversal

The macros for resuming list traversal must also be used within an RCU read-side critical section. It is the caller's responsibility to make sure that the list element where traversal has started remains in existence, and the usual way to do this is to make sure that the same RCU read-side critical section covers both the original traversal and the resumption. 

The individual API members are as follows: 

  * `list_for_each_entry_continue_rcu()`: Iterate over an RCU-protected list from the specified element. 
  * `hlist_for_each_entry_continue_rcu()`: Iterate over an RCU-protected hlist from the specified element. 
  * `hlist_for_each_entry_continue_rcu_bh()`: Iterate over an RCU-bh-protected hlist from the specified element. 

#####  Stepwise Traversal

The macros for doing stepwise traversal are used when more control is required. When repeatedly invoking these macros to step through a list, the full set of macro invocations must be enclosed in an RCU read-side critical section. If you must split the traversal into more than one RCU read-side critical section, you must also do something else to guarantee the existence of the relevant elements during the time between successive RCU read-side critical sections. 

The individual API members are as follows: 

  * `list_entry_rcu()`: Given a pointer to a `list_head` structure, return a pointer to the enclosing element. 
  * `list_first_or_null_rcu()`: Return a pointer to the first element on an RCU-protected list, or `NULL` if the list is empty. 
  * `list_next_rcu()`: Return a pointer to the next element on the list, which must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_first_rcu()`: Return a pointer to the first element on an RCU-protected hlist, or `NULL` if the hlist is empty. If non-NULL, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_next_rcu()`: Return a pointer to the next element on an RCU-protected hlist, or `NULL` if at the end of the hlist. If non-NULL, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_pprev_rcu()`: Return a pointer to the previous element's pointer to the current element. This must not be used by RCU readers because the ->pprev pointer is poisoned at deletion time. 
  * `hlist_nulls_first_rcu()`: Return a pointer to the first element on an RCU-protected hlist-nulls, or `NULL` if the hlist is empty. If non-NULL, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_nulls_next_rcu()`: Return a pointer to the next element on an RCU-protected hlist-nulls, or `NULL` if at the end of the hlist. If non-NULL, it must be handed to one of the `rcu_dereference()` family of primitives if used by an RCU reader. 
  * `hlist_bl_first_rcu()`: Return a pointer to the first element on the bitlocked hlist, applying `rcu_dreference()`. The caller must either be in an RCU read-side critical section or have locked the hlist. Note that this cannot be an RCU-bh, RCU-sched, or SRCU read-side critical section, only an RCU read-side critical section will do. 

#####  List Addition

The list-addition APIs require some form of mutual exclusion, for example, using locking or single designated updater task. 

The individual API members are as follows: 

  * `list_add_rcu()`: Add an element at the head of the RCU-protected list. 
  * `list_add_tail_rcu()`: Add an element at the tail of the RCU-protected list. 
  * `hlist_add_after_rcu()`: Add an element after the specified element in the RCU-protected hlist. 
  * `hlist_add_before_rcu()`: Add an element before the specified element in the RCU-protected hlist. 
  * `hlist_add_head_rcu()`: Add an element to the beginning of the RCU-protected hlist. 
  * `hlist_nulls_add_head_rcu()`: Add an element to the beginning of the RCU-protected hlist-nulls. 
  * `hlist_bl_add_head_rcu()`: Add an element to the beginning of the RCU-protected bitlocked hlist. 
  * `hlist_bl_set_first_rcu()`: Add an element to the beginning of the RCU-protected bitlocked hlist, which must initially be empty. 

#####  List Deletion

The list-deletion APIs also require some form of mutual exclusion, for example, using locking or a single designated updater task. 

The individual API members are as follows: 

  * `list_del_rcu()`: Delete the specified element from its RCU-protected list. 
  * `hlist_del_rcu()`: Delete the specified element from its RCU-protected hlist. 
  * `hlist_del_init_rcu()`: Delete the specified element from its RCU-protected hlist, initializing its pointer to form an empty list. 
  * `hlist_nulls_del_rcu()`: Delete the specified element from its RCU-protected hlist-nulls. 
  * `hlist_nulls_del_init_rcu()`: Delete the specified element from its RCU-protected hlist-nulls, initializing its pointer to form an empty list. 
  * `hlist_bl_del_rcu()`: Delete the specified element from its RCU-protected bitlocked hlist. 
  * `hlist_bl_del_init_rcu()`: Delete the specified element from its RCU-protected bitlocked hlist, initializing its pointer to form an empty list. 

#####  List Replacement

The list-replacement APIs replace an existing element with a new version. The caller is responsible for disposing of the old (existing) element. These also require some form of mutual exclusion, for example, using locking or a single designated updater task. 

The individual API members are as follows: 

  * `list_replace_rcu()`: Replace an element in an RCU-protected list. 
  * `hlist_replace_rcu()`: Replace an element in an RCU-protected hlist. 

#####  List Splice

The list-splice APIs join a pair of lists into a single combined list. This also requires some form of mutual exclusion, for example, using locking or a single designated updater task. 

The sole API member is as follows: 

  * `list_splice_init_rcu()`: Splice a pair of lists together, initializing the source list to empty. Note that this primitive waits for a grace period to elapse. You determine the RCU flavor by passing in the corresponding grace-period-wait primitive, for example, `synchronize_rcu()` for RCU and `synchronize_rcu_bh()` for RCU-bh. 

####  Kernel configuration parameters

RCU's kernel configuration parameters can be considered to be part of the RCU API, most especially from the viewpoint of someone building a kernel intended for a specialized device or workload. This section summarizes the RCU-related configuration parameters. 

The first set of Kconfig parameters controls the underlying behavior of the RCU implementation itself, and is defined in `init/Kconfig`. 

  * `CONFIG_PREEMPT=n` and `CONFIG_SMP=y` implies `CONFIG_TREE_RCU`, selecting the non-preemptible tree-based RCU implementation that is appropriate for server-class SMP builds. It can accommodate a very large number of CPUs, but scales down sufficiently well for all but the most memory-constrained systems. `CONFIG_TREE_RCU` provides the following boot parameters: 
    * `rcutree.blimit=` sets the maximum number of RCU callbacks to process in one batch, which defaults to ten callbacks. This limit does not apply to offloaded CPUs. 
    * `rcutree.qhimark=` sets the threshold of queued RCU callbacks beyond which `rcutree.blimit` will be ignored. This defaults to 10,000 callbacks. 
    * `rcutree.qlowmark=` sets the threshold of queued RCU callbacks below which `rcutree.blimit` will once again have effect. This defaults to 100 callbacks. 
    * `rcutree.jiffies_till_first_fqs=` sets the number of jiffies to wait between grace-period initialization and the first force-quiescent-state scan that checks (among other things) for idle CPUs. The default depends on the value of `HZ` and the number of CPUs on the system. All systems wait for at least one jiffy, with one additional jiffy for `HZ` greater than 250, an additional jiffy for `HZ` greater than 500, and one additional jiffy for each 256 CPUs on the system. 
    * `rcutree.jiffies_till_next_fqs=` sets the number of jiffies to wait between successive force-quiescent-state scans. The default is the same as for `rcutree.jiffies_till_first_fqs`. 
    * `rcutree.jiffies_till_sched_qs=` sets the number of jiffies that a grace period will wait before soliciting help from `rcu_note_context_switch()`, which will involve sending IPIs to the holdout CPUs. 
    * `rcupdate.rcu_expedited=` causes normal grace-period primitives to act like their expedited counterparts. For example, invoking `synchronize_rcu()` will act as if `synchronize_rcu_expedited()` had been invoked. 
  * `CONFIG_PREEMPT=y` implies `CONFIG_TREE_PREEMPT_RCU`, selecting the preemptible tree-based RCU implementation that is appropriate for real-time and low-latency SMP builds. It can also accommodate a very large number of CPUs, and also scales down sufficiently well for all but the most memory-constrained systems. The boot parameters for `CONFIG_TREE_RCU` also apply to `CONFIG_TREE_PREEMPT_RCU`. 
  * `CONFIG_PREEMPT=n` and `CONFIG_SMP=n` implies `CONFIG_TINY_RCU`, selecting the non-preemptible uniprocessor RCU implementation that is appropriate for non-real-time UP builds. It has the smallest memory footprint of any of the current in-kernel RCU implementations. In fact, its memory footprint is so small that it doesn't even have any kernel boot parameters. 

**Quick Quiz 8** : Why doesn't the `rcupdate.rcu_expedited=` boot parameter also apply to `CONFIG_TINY_RCU`?   
Answer

The second set of Kconfig parameters controls RCU's energy-efficiency features. These are also defined in `init/Kconfig`. 

  * `CONFIG_RCU_FAST_NO_HZ=y` improves RCU's energy efficiency by reducing the number of times that RCU wakes up idle CPUs. The downside of this approach is that it increases RCU grace-period latency somewhat. 
    * `rcutree.rcu_idle_gp_delay=` specifies the number of jiffies an idle CPU with callbacks should remain idle before rechecking RCU state. The default is four jiffies. 
    * `rcutree.rcu_idle_lazy_gp_delay=` specifies the number of jiffies an idle CPU with callbacks, where all callbacks are lazy, should remain idle before rechecking RCU state. (A "lazy" callback is one that RCU knows will do nothing other than free memory.) The default is six seconds, or `6*HZ` jiffies. 
  * `CONFIG_RCU_NOCB_CPU=y` [fortuitously improves RCU's energy efficiency [PDF]](<http://www2.rdrop.com/users/paulmck/realtime/paper/AMPenergy.2013.04.19a.pdf>) by eliminating wakeups due to RCU callback processing. However, it was intended for real-time use, so it is covered in the next section. 

Please note that these features do not pertain to `CONFIG_TINY_RCU`, whose job description emphasizes small memory footprint over energy efficiency. 

The third set of Kconfig parameters controls RCU's real-time features, which are also defined in `init/Kconfig`. 

  * `CONFIG_RCU_NOCB_CPU=y` allows callback processing to be [offloaded from selected CPUs](<https://lwn.net/Articles/522262/>) ("NOCB" stands for "no callbacks") onto `rcuo` kthreads. The CPUs to offload can be specified at boot time, or by one of the following Kconfig parameters, all of which depend on `CONFIG_RCU_NOCB`: 

    * `CONFIG_RCU_NOCB_CPU_NONE=y`: No CPUs will be offloaded unless specified at boot time. 
    * `CONFIG_RCU_NOCB_CPU_ZERO=y`: CPU 0 will be offloaded, and other CPUs can be specified as offloaded at boot time. 
    * `CONFIG_RCU_NOCB_CPU_ALL=y`: All CPUs will be offloaded. 

The following kernel boot parameters also apply to callback offloading: 
    * `rcu_nocbs=` may be used to specify offloaded CPUs at boot time. For example, `rcu_nocbs=1-3,7` would cause CPUs 1, 2, 3, and 7 to have their callbacks offloaded to `rcuo` kthreads. The set of offloaded CPUs cannot be changed at runtime: In all such cases thus far, offloading all CPUs has sufficed. 
    * `rcu_nocb_poll=` also offloads the need to do wakeup operations from the offloaded CPUs. On the other hand, this means that all of the `rcuo` kthreads must poll, which probably is not what you want on a battery-powered system. 
    * `rcutree.rcu_nocb_leader_stride=` sets the number of NOCB kthread groups, which defaults to the square root of the number of CPUs. Larger numbers reduces the wakeup overhead on the per-CPU grace-period kthreads, but increases that same overhead on each group's leader. 
  * `CONFIG_NO_HZ_FULL=y` implies `CONFIG_RCU_USER_QS`, which causes RCU to treat userspace execution as an extended quiescent state similar to RCU's handling of dyntick-idle CPUs. 
  * `CONFIG_RCU_BOOST=y` enables RCU priority boosting. This could be considered a debugging option, but it is one that pertains primarily to real-time kernels, so is included in the real-time section. This Kconfig parameter causes blocked RCU readers to be priority-boosted in order to avoid indefinite prolongation of the current RCU grace period. The following Kconfig parameters control the boosting process: 

    * `CONFIG_RCU_BOOST_PRIO=` specifies the real-time priority to boost to, and defaults to priority one, the least-important real-time priority level. You should set this priority level to be greater than the highest-priority real-time CPU-bound thread. The default priority is appropriate for the common case where there are no CPU-bound threads running at real-time priority. 
    * `CONFIG_RCU_BOOST_DELAY=` specifies how long RCU will allow a grace period to be delayed before starting RCU priority boosting. The default is 300 milliseconds, which seems to work quite well in practice.  

Please note that these Kconfig options do not pertain to `CONFIG_TINY_RCU`, which again is focused on small memory footprint, even at the expense of real-time response. 

The fourth set of Kconfig parameters may also be specified to tune the data-structure layout of `CONFIG_TREE_RCU` and `CONFIG_TREE_PREEMPT_RCU`: 

  * `CONFIG_RCU_FANOUT=` controls the fanout of non-leaf nodes of the tree. Lower fanout values reduce lock contention, but also consume more memory and increase the overhead of grace-period computations. The default values have always sufficed with the exception of RCU stress testing. 
  * `CONFIG_RCU_FANOUT_LEAF=` controls the fanout of leaf nodes of the tree. As for `CONFIG_RCU_FANOUT`, lower fanout values reduce lock contention, but also consume more memory and increase the overhead of grace-period computations. The default values are sufficient in most cases, but very large systems (thousands of CPUs) will want to set this to the largest possible value, namely 64. Such systems will also need to boot with skew_tick=1 to avoid massive lock contention on the leaf `rcu_node` `->lock` fields. This fanout can also be set at boot time: 

    * `rcutree.rcu_fanout_leaf=` sets the number of CPUs to assign to each leaf-level `rcu_node` structure. This defaults to 16 CPUs. Very large systems (many hundreds of CPUs) can benefit from setting this to its maximum (64 on 64-bit systems), but such systems should also set `skew_tick=1`. 
  * `CONFIG_RCU_FANOUT_EXACT=y` forces the tree to be as balanced as possible. Again, to the best of my knowledge, the default values have always been sufficient. 

The fifth set of kernel configuration parameters controls debugging options: 

  * `CONFIG_RCU_TRACE=y` enables `debugfs`-based tracing as well as event tracing. In `CONFIG_TINY_RCU` kernels, it also enables RCU CPU stall warnings; see `CONFIG_RCU_CPU_STALL_TIMEOUT` below for more details. 
  * `CONFIG_SPARSE_RCU_POINTER=y` enables sparse-based checks of proper use of RCU-protected pointers. Please note that this is a build-time check: use "make C=1" to cause sparse to check source files that would have been rebuilt by "make", and use "make C=2" to cause sparse to unconditionally check source files. 
  * `CONFIG_DEBUG_OBJECTS_RCU_HEAD=y` enables debug-objects checking of multiple invocations of `call_rcu()` (and friends) on the same structure. 
  * `CONFIG_PROVE_RCU=y` enables lockdep-RCU checking. If `CONFIG_PROVE_RCU_REPEATEDLY` is also specified, then the lockdep-RCU checking can output multiple lockdep-RCU "splats", otherwise only a single lockdep-RCU splat will be emitted per boot. 
  * `CONFIG_RCU_TORTURE_TEST=y` or `CONFIG_RCU_TORTURE_TEST=m` enables RCU torture testing. This is a tri-state parameter, permitting `rcutorture.c` to be compiled into the kernel, built as a module, or omitted entirely. When `rcutorture.c` is built into the kernel (`CONFIG_RCU_TORTURE_TEST=y`), then `CONFIG_RCU_TORTURE_TEST_RUNNABLE` starts RCU torture testing during boot. Please don't try this on production systems. 
  * `CONFIG_RCU_CPU_STALL_TIMEOUT=` specifies the maximum grace-period duration that RCU will tolerate without complaint. Excessively long grace periods are usually caused by CPUs or tasks failing to find their way out of an RCU read-side critical section in a timely manner. CPU stalls can be caused by a number of bugs, as described in `Documentation/RCU/stallwarn.txt`. This Kconfig variable defaults to 21 seconds. 

    * `rcupdate.rcu_cpu_stall_suppress=` suppresses RCU CPU stall warning messages. 
    * `rcupdate.rcu_cpu_stall_timeout=` overrides the build-time `CONFIG_RCU_CPU_STALL_TIMEOUT` setting. 
  * `CONFIG_RCU_CPU_STALL_VERBOSE=y` causes detailed per-task information to be printed when a CPU stall is encountered. 
  * `CONFIG_RCU_CPU_STALL_INFO=y` prints out additional per-CPU diagnostic information when a CPU stall is encountered. 

If you are working with code that uses RCU, please do us all a favor and test that code with `CONFIG_PROVE_RCU` and `CONFIG_DEBUG_OBJECTS_RCU_HEAD` enabled. Please also consider running sparse with `CONFIG_SPARSE_RCU_POINTER`. If you are modifying the RCU implementation itself, you will need to run rcutorture, with multiple runs covering the relevant kernel configuration parameters. A one-hour rcutorture run on an 8-CPU machine qualifies as light rcutorture testing. The automated scripts invoked by `tools/testing/selftests/rcutorture/bin/kvm.sh` can be quite helpful. 

Yes, running extra tests can be a hassle, but I am here to tell you that extra testing is _much_ easier than trying to track down bugs in your RCU code. 

####  How did those 2010 predictions turn out?

How good were those old [predictions](<https://lwn.net/Articles/418853/#What Next for the RCU API?>)? 

  * "Complete implementation of RCU priority boosting (`TINY_RCU` submission slated for 2.6.38, `TREE_RCU` implementation in progress)."

RCU priority boosting is now in mainline, although the `TINY_PREEMPT_RCU` implementation is now [gone entirely](<https://lwn.net/Articles/541037/>). Uniprocessor preemptible kernels now use `TREE_PREEMPT_RCU`. Uniprocessor non-preemptible kernels still use `TINY_RCU` for its small memory footprint. 

  * "Support for running certain types of user applications without scheduler ticks. There was a spirited discussion on this topic at Linux Plumbers Conference, but at this writing it appears that only small tweaks to RCU will be required."

This [work](<https://lwn.net/Articles/549580/>) is mostly completed, with many thanks to Frédéric Weisbecker for doing much of the heavy lifting. The RCU work included unplanned support for callback offloading and [energy efficiency](<https://lwn.net/Articles/558284/>). 

  * "Merge the implementation of SRCU into `TINY_RCU` and `TREE_RCU`. A design for this is in mostly in place. This effort is likely to result in `call_srcu()` and `srcu_barrier()`. If it does, please be _very_ careful with these primitives!!!"

This never happened. Instead, SRCU was rewritten from scratch by Peter Zijlstra, Lai Jiangshan, and myself, with Lai's implementation being accepted into the kernel. That said, you should still be very careful with the new `call_srcu()` and `srcu_barrier()` primitives. 

  * "Make `RCU_FAST_NO_HZ` work for `TREE_PREEMPT_RCU`."

This work was completed. In fact, the `RCU_FAST_NO_HZ` code was reworked several times, culminating in the current implementation, which actually provides measurable energy savings on real hardware. 

  * "Drive the choice between `TINY_PREEMPT_RCU`, `TINY_RCU`, `TREE_PREEMPT_RCU`, and `TREE_RCU` entirely off of `SMP` and `PREEMPT`. This would allow cutting code and test scenarios, but first `TINY_PREEMPT_RCU` must prove itself."

This is done. Now only the most dedicated kernel hackers manually choose the RCU implementation, editing the Kconfig files to impose their will on the kernel. 

  * "It is possible that `rcu_dereference_index_check()` will be retired if it is reasonable to convert all current use of RCU-protected indexes into RCU-protected pointers."

Nope, `rcu_dereference_index_check()` is still with us. 

  * "It is quite possible that large systems might encounter problems with `synchronize_rcu_expedited()` scalability."

There have been complaints about the realtime latency impact of starting new normal grace periods (problem described [here [PDF]](<http://www2.rdrop.com/users/paulmck/realtime/paper/bigrt.2012.07.10a.pdf>), fix described [here](<https://lwn.net/Articles/518953/>)), but no complaints about the scalability of `synchronize_rcu_expedited()`, so no scalability work on expedited grace periods has been done. 

  * "Make RCU be more aggressive about entering dyntick-idle state when running on lightly loaded systems with four or more CPUs."

The aforementioned `RCU_FAST_NO_HZ` and callback-offloading work has done just this. 

So, as predictions go, they weren't too bad. :-) 

It is also illuminating to list the unexpected changes. Some of these are hinted at above, but bear repeating: 

  * Getting rid of `TINY_PREEMPT_RCU`. 
  * Callback offloading. 
  * Although the need for increased energy efficiency was no surprise, the need for energy efficiency while running applications without scheduler-clock ticks certainly was. 
  * The need for SRCU to be usable in the idle loop and from offline CPUs. 
  * The need for deep sub-millisecond realtime response on systems with 4096 CPUs. 
  * The need to handle interrupts that never return, as well as the need to return from interrupts that never happened. 
  * The fact that the RCU CPU stall warning code needed to be written with just as much careful attention to concurrency as the rest of RCU. My attitude in 2010 was that if the stall had been proceeding for a full 60 seconds, there was no need to be dainty. I have spent much time in the intervening four years learning, one bug at a time, just how much daintiness is required. 
  * The heavy use of RCU from within the idle loop, which resulted in the `RCU_NONIDLE()` macro and the `_rcuidle` suffix for event tracing. 
  * That I would have eliminated all day-one bugs from RCU. Of course, the way I eliminated all day-one bugs was to eliminate all the day-one code, which some might consider to be cheating. 

####  What next for the RCU API?

The most honest answer is that I do not know. The next steps for the RCU API will be decided as they always have been, by the needs of RCU's users and by the limits of the technology at the time. That said, the following seem to be a few of the more likely directions: 

  * It is possible that `rcu_dereference_index_check()` will be retired if it is reasonable to convert all current use of RCU-protected indexes into RCU-protected pointers. Yes, I am doubling down on this one. 

  * It is quite possible that large systems might encounter problems with `synchronize_rcu_expedited()` scalability. I am doubling down on this one as well, and extending it to normal grace-period operations. For example, it might be necessary to parallelize grace-period operations. For another example, it might be necessary to make `synchronize_rcu_expedited()` stop interrupting dyntick-idle CPUs. 

  * Additional diagnostics will be required, for example, detecting pointers leaking out of RCU read-side critical sections. 

  * A `kmem_struct` counterpart to `kfree_rcu()` will likely be required. 

  * Inlining of `TREE_PREEMPT_RCU`'s `rcu_read_lock()` primitive. 

But if the past is any guide, new use cases and workloads will place unanticipated demands on RCU. 

#### Acknowledgments

We are all indebted to a huge number of people who have used, abused, poked at, and otherwise helped to improve the RCU API. I am grateful to Jim Wasko for his support of this effort. 

####  Answers to Quick Quizzes

**Quick Quiz 1** : Why is extreme caution required for `call_srcu()` and `srcu_barrier()`? 

**Answer** : Because SRCU readers are allowed to block indefinitely, these two primitives might take a long time to return, if they return at all. So if you use either `call_srcu()` or `srcu_barrier()`, it is your responsibility to make sure that readers complete in a timely fashion, lest large numbers of `call_srcu()` calls OOM the kernel or your `srcu_barrier()` refuse to ever return. Or, alternatively, it is your responsibility to make sure that your code does not care that `call_srcu()` and `srcu_barrier()` take forever to find the end of the grace period. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : If `kfree_rcu()` is so popular, why isn't there a `kfree_rcu_bh()`, `kfree_rcu_sched()`, or `kfree_srcu()`? For that matter, why not a `kmem_cache_free_rcu()`? 

**Answer** : Because there are a lot fewer uses of `call_rcu_bh()`, `call_rcu_sched()`, and `call_srcu()` than there are of `call_rcu()`, there was much greater motivation for `kfree_rcu()` than for the others. In particular, there are only about four uses of `call_rcu_bh()` that could instead use a `kfree_rcu_bh()`, only one use of `call_rcu_sched()` that could instead use a `kfree_rcu_sched()`, and no uses of `call_srcu()` that could instead use a `kfree_srcu()`. Should any of these reach ten uses, it might be time to introduce the corresponding `kfree_`-style primitive. 

A `kmem_cache_free_rcu()` might make sense given a sufficiently simple and fast implementation. There are not yet enough uses for any of `kmem_cache_free_rcu_bh()`, `kmem_cache_free_rcu_sched()`, or `kmem_cache_free_srcu()` to be worth adding. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : What happens if you mix and match RCU and RCU-Sched? 

**Answer** : In a `CONFIG_TREE_RCU` or a `CONFIG_TINY_RCU` kernel, mixing these two works "by accident" because in those kernel builds, RCU and RCU-Sched map to the same implementation. However, this mixture is fatal in `CONFIG_TREE_PREEMPT_RCU` builds, due to the fact that RCU's read-side critical sections can then be preempted, which would permit `synchronize_sched()` to return before the RCU read-side critical section reached its `rcu_read_unlock()` call. This could, in turn, result in a data structure being freed before the read-side critical section was finished with it, which could, in turn, greatly increase the actuarial risk experienced by your kernel. 

Even in `CONFIG_TREE_RCU` and `CONFIG_TINY_RCU` builds, such mixing and matching is of course very strongly discouraged. Mixing and matching other flavors of RCU is even worse: it can result in hard-to-debug bad-pointer bugs. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : Can `synchronize_srcu()` be safely used within an SRCU read-side critical section? If so, why? If not, why not? 

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

**Back to Quick Quiz 4**.

**Quick Quiz 5** : Why isn't there an `smp_mb__after_rcu_read_unlock()`, `smp_mb__after_rcu_bh_read_unlock()`, or `smp_mb__after_rcu_sched_read_unlock()`? 

**Answer** : Because these primitives never imply any sort of barrier. In contrast, the current implementation of `srcu_read_unlock()` actually does imply a full barrier, so `smp_mb__after_srcu_read_unlock()` can be an informative no-op. 

**Back to Quick Quiz 5**.

**Quick Quiz 6** : Why would anyone need to distinguish lists based on their NULL pointers? Why not just remember which list you started searching??? 

**Answer** : Suppose that CPU 0 is traversing such a list within an RCU read-side critical section, where the elements are allocated from `SLAB_DESTROY_BY_RCU` slab cache. The elements could therefore be freed and reallocated at any time. If CPU 0 is referencing an element while CPU 1 is freeing that element, and if CPU 1 then quickly reallocates that same element and adds it to some other list, then CPU 0 will be transported to that new list along with the element. In this case, remembering the starting list would clearly be unhelpful. 

To make matters worse, suppose that CPU 0 searches a list and fails to find the element that it was looking for. Was that because the element did not exist? Or because CPU 0 got transported to some other list in the meantime? Readers traversing `SLAB_DESTROY_BY_RCU` lists must carefully validate each element and check for being moved to another list. One way to check for being moved to another list is for each list to have its own value for the `NULL` pointer. These checks are subtle and easy to get wrong, so please be careful. 

**Back to Quick Quiz 6**.

**Quick Quiz 7** : Why is there no `hlist_nulls_add_tail_rcu()`? 

**Answer** : Suppose that CPU 0 is traversing an hlist-nulls list under RCU protection. Suppose that while CPU 0 is referencing list element A, CPU 1 frees it and reallocates it, adding it to another list, which CPU 0 unwittingly starts traversing. Suppose further that while CPU 0 is referencing an element B in the new list, CPU 2 frees and reallocates it, moving it back to the original list. When CPU 0 comes to the end of the original list, it sees that the `NULL` pointer has the proper value, so does not realize that it has been moved. 

If CPU 2 had added element B at the tail of the list, CPU 0 would be within its rights to conclude that it had fully searched this list when in fact it had not. But given that it is only possible to add elements to the head of an hlist-nulls list, any CPU coming to the end of the same list it started traversing can be sure that it really did search the entire list. Possibly several times, if it was extremely unlucky. 

Therefore, there is not and should never be a primitive to add to the middle or the end of an RCU-protected hlist-nulls list. Except maybe at initialization time, before any readers have access to the list. 

**Back to Quick Quiz 7**.

**Quick Quiz 8** : Why doesn't the `rcupdate.rcu_expedited=` boot parameter also apply to `CONFIG_TINY_RCU`? 

**Answer** : Because `CONFIG_TINY_RCU`'s normal grace-period primitives are no-ops, they are already maximally expedited. Nothing is faster than doing nothing. 

**Back to Quick Quiz 8**.
