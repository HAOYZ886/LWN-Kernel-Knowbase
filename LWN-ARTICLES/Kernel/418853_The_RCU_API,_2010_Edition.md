---
title: "The RCU API, 2010 Edition"
url: https://lwn.net/Articles/418853/
date: "December 8, 2010"
category: "Read-copy-update"
author: "December 8, 2010 This article was contributed by Paul McKenney"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

December 8, 2010

This article was contributed by Paul McKenney

#### Introduction

Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October of 2002. RCU is most frequently described as a replacement for reader-writer locking, but has also been used in a number of other ways. RCU is notable in that RCU readers do not directly synchronize with RCU updaters, which makes RCU read paths extremely fast, and also permits RCU readers to accomplish useful work even when running concurrently with RCU updaters. 

Although the basic idea behind RCU has not changed in decades following its introduction into DYNIX/ptx, the RCU API has evolved significantly even over the past three years, most recently due to software-engineering concerns. This evolution is documented by the following sections. 

  1. Software-Engineering Enhancements
  2. RCU has a Family of Wait-to-Finish APIs
  3. RCU has List-Based Publish-Subscribe and Version-Maintenance APIs
  4. RCU has Pointer-Based Publish-Subscribe and Version-Maintenance APIs
  5. RCU has Debugging APIs
  6. Kernel Configuration Parameters
  7. What Next for the RCU API?

These sections are followed by answers to the Quick Quizzes. 

####  Software-Engineering Enhancements

Pre-Linux experience with RCU-like primitives featured either a small number of uses, a small number of developers, or both. That experience gave no reason to believe that Linux would ever contain more than a few hundred RCU call sites which could easily be validated by inspection on each release of the Linux kernel, thus making computer-assisted validation unnecessary. Nevertheless, Linux developers quickly added checking in the form of "scheduling while atomic" that triggered if a thread blocked within an RCU read-side critical section in `CONFIG_PREEMPT` kernels. This provided as much checking as did any previous RCU implementation, so life was good, at least for a while. 

Around 2005, the uptake of RCU in the Linux kernel increased substantially. Where it took about four years for the first thousand RCU call sites, the second thousand took only about 18 months, as did the third thousand. Even given only the three thousand RCU call sites that were in the kernel at the beginning of 2010, rechecking each instance during the week after each release's merge window would leave less than two minutes for each instance, assuming an 80-hour week. The odds of spotting subtle bugs on that kind of schedule are vanishingly small. This point was underscored by a [patchset](<http://lkml.org/lkml/2009/12/9/479>) from Thomas Gleixner fixing a number of `rcu_read_lock()`-omission bugs; Tetsuo Handa [followed up with a list of no fewer than 47 similar bugs](<http://lkml.org/lkml/2009/12/10/496>). 

I therefore started work on a patch that used [lockdep](<http://lwn.net/Articles/185666/>) to ensure that `rcu_dereference()` calls are properly protected, either by an RCU read-side critical section, by the update-side lock, or by being inaccessible to readers, as [reported on LWN](<https://lwn.net/Articles/371986/>). I naively expected this to be the end of the story, but experience with problems reported by lockdep RCU resulted in a number of changes to the RCU API, which are documented below. 

Mathieu Desnoyers took interest in another set of RCU-usage bugs in which the developer passes a given structure to `call_rcu()` twice within a given grace period, and submitted a [patch](<http://lkml.org/lkml/2010/3/18/417>) that uses debug objects to detect this sort of abuse. Because the debug-objects facility does not track on-stack variables, Mathieu added an additional pair of RCU APIs to control tracking of on-stack `rcu_head` structures. 

Arnd Bergmann noted that lockdep RCU cannot be expected to locate bugs where an RCU-protected pointer is accessed directly, without the benefit of `rcu_dereference()`. He therefore enlisted the aid of the [sparse](<https://sparse.wiki.kernel.org/index.php/Main_Page>) static analysis tool by annotating declarations of RCU-protected pointers with a new `__rcu` macro. This macro allows sparse to complain when an RCU-protected pointer is accessed without the assistance of one of the `rcu_dereference()` family of RCU APIs, which required some modifications to those APIs. 

A few other RCU API changes resulted from usage, filling out the API as the need for additional RCU primitives arose. The next sections discuss aspects of the RCU API, highlighting recent changes. 

####  RCU has a Family of Wait-to-Finish APIs

The most straightforward answer to "what is RCU" is that RCU is an API used in the Linux kernel, as summarized by [the big API table](<https://lwn.net/Articles/419086/>) and the following discussion. Or, more precisely, RCU is a four-member family of APIs as shown in the table, with each column corresponding to one of the family members. 

If you are new to RCU, you might consider focusing on just one of the columns in the big RCU API table. For example, if you are primarily interested in understanding how RCU is most frequently used in the Linux kernel, "RCU" would be the place to start. On the other hand, if you want to understand RCU for its own sake, "SRCU" has the simplest API. You can always come back to the other columns later. If you are already familiar with RCU, this table can serve as a useful reference. 

**Quick Quiz 1** : Why are some of the cells in the big table colored green? 

**Quick Quiz 2** : Why are some of the cells in the big table colored blue? 

The "RCU" column corresponds to the original RCU implementation, in which RCU read-side critical sections are delimited by `rcu_read_lock()` and `rcu_read_unlock()`, which may be nested. The corresponding synchronous update-side primitives, `synchronize_rcu()`, along with its synonym `synchronize_net()`, wait for any currently executing RCU read-side critical sections to complete. The length of this wait is known as a "grace period". If grace periods are too long for you, `synchronize_rcu_expedited()` speeds things up by about an order of magnitude, but at the expense of significant CPU overhead. The asynchronous update-side primitive, `call_rcu()`, invokes a specified function with a specified argument after a subsequent grace period. For example, `call_rcu(p,f);` will result in the "RCU callback" `f(p)` being invoked after a subsequent grace period. There are situations, [ such as when unloading a module that uses `call_rcu()`](<http://lwn.net/Articles/217484/>), when it is necessary to wait for all outstanding RCU callbacks to complete. The `rcu_barrier()` primitive does this job. 

In the "RCU BH" column, `rcu_read_lock_bh()` and `rcu_read_unlock_bh()` delimit RCU read-side critical sections, and `call_rcu_bh()` invokes the specified function and argument after a subsequent grace period. There are also `synchronize_rcu_bh()`, `synchronize_rcu_bh_expedited()`, and `rcu_barrier_bh()` primitives, which are analogous to their "RCU" counterparts. 

In the "RCU Sched" column, anything that disables preemption acts as an RCU read-side critical section, and `synchronize_sched()` waits for the corresponding RCU grace period. This RCU API family was added in the 2.6.12 kernel, which split the old `synchronize_kernel()` API into the current `synchronize_rcu()` (for RCU) and `synchronize_sched()` (for RCU Sched). There are also `synchronize_sched()`, `synchronize_sched_expedited()`, and `rcu_barrier_sched()` primitives, which are analogous to their "RCU" counterparts. 

**Quick Quiz 3** : What happens if you mix and match RCU and RCU Sched? 

The "SRCU" column displays a specialized RCU API that permits general sleeping in RCU read-side critical sections, as was described in the LWN article [Sleepable RCU](<http://lwn.net/Articles/202847/>). Of course, use of `synchronize_srcu()` in an SRCU read-side critical section can result in self-deadlock, so should be avoided. SRCU differs from earlier RCU implementations in that the caller allocates an `srcu_struct` for each distinct SRCU usage. This approach prevents SRCU read-side critical sections from blocking unrelated `synchronize_srcu()` invocations. In addition, in this variant of RCU, `srcu_read_lock()` returns a value that must be passed into the corresponding `srcu_read_unlock()`. 

**Quick Quiz 4** : Why does SRCU lack an asynchronous `call_srcu()` interface? 

**Quick Quiz 5** : Can `synchronize_srcu()` be safely used within an SRCU read-side critical section? If so, why? If not, why not? 

The Linux kernel currently has a surprising number of RCU APIs and implementations. There is some hope of reducing this number, but careful inspection and analysis will be required before removing either an implementation or any API members, just as would be required before removing one of the many locking APIs in the Linux kernel. 

####  RCU has List-Based Publish-Subscribe and Version-Maintenance APIs

Fortunately, most of RCU's list-based publish-subscribe and version-maintenance primitives shown in the following table apply to all of the variants of RCU discussed above. This commonality can in some cases allow more code to be shared, which certainly reduces the API proliferation that would otherwise occur. However, it is quite likely that software-engineering considerations will eventually result in variants of these list-handling primitives that are specialized for each given flavor of RCU. 

Category | Primitives | Purpose  
---|---|---  
List traversal | `list_for_each_entry_rcu()` | Iterate over an RCU-protected list from the beginning.  
`list_for_each_entry_continue_rcu()` | Iterate over an RCU-protected list from the specified element.  
`list_entry_rcu()` | Given a pointer to a raw `list_head` in an RCU-protected list, return a pointer to the enclosing element.  
`list_first_entry_rcu()` | Return the first element of an RCU-protected list.  
List update | `list_add_rcu()` | Add an element to the head of an RCU-protected list.  
`list_add_tail_rcu()` | Add an element to the tail of an RCU-protected list.  
`list_del_rcu()` | Delete the specified element from an RCU-protected list, poisoning the `->pprev` pointer but not the `->next` pointer.  
`list_replace_rcu()` | Replace the specified element in an RCU-protected list with the specified element.  
`list_splice_init_rcu()` | Move all elements from an RCU-protected list to another RCU-protected list.  
Hlist traversal | `hlist_for_each_entry_rcu()` | Iterate over an RCU-protected hlist from the beginning.  
`hlist_for_each_entry_rcu_bh()` | Iterate over an RCU-bh-protected hlist from the beginning.  
`hlist_for_each_entry_continue_rcu()` | Iterate over an RCU-protected hlist from the specified element.  
`hlist_for_each_entry_continue_rcu_bh()` | Iterate over an RCU-bh-protected hlist from the specified element.  
Hlist update | `hlist_add_after_rcu()` | Add an element after the specified element in an RCU-protected hlist.  
`hlist_add_before_rcu()` | Add an element before the specified element in an RCU-protected hlist.  
`hlist_add_head_rcu()` | Add an element at the head of an RCU-protected hlist.  
`hlist_del_rcu()` | Delete the specified element from an RCU-protected hlist, poisoning the `->pprev` pointer but not the `->next` pointer.  
`hlist_del_init_rcu()` | Delete the specified element from an RCU-protected hlist, initializing the element's reverse pointer after deletion.  
`hlist_replace_rcu()` | Replace the specified element in an RCU-protected hlist with the specified element.  
Hlist nulls traversal | `hlist_nulls_for_each_entry_rcu()` | Iterate over an RCU-protected hlist-nulls list from the beginning.  
Hlist nulls update | `hlist_nulls_del_init_rcu()` | Delete the specified element from an RCU-protected hlist-nulls list, initializing the element after deletion.  
`hlist_nulls_del_rcu()` | Delete the specified element from an RCU-protected hlist-nulls list, poisoning the `->pprev` pointer but not the `->next` pointer.  
`hlist_nulls_add_head_rcu()` | Add an element to the head of an RCU-protected hlist-nulls list.  
  
The first pair of categories operate on the Linux kernel's `struct list_head` lists, which are circular, doubly-linked lists. These primitives permit lists to be modified in the face of concurrent traversals by readers. The list-traversal primitives are implemented with simple instructions, so are extremely lightweight, though they also [execute a memory barrier on DEC Alpha](<http://www.openvms.compaq.com/wizard/wiz_2637.html>). The list-update primitives that add elements to a list incur memory-barrier overhead, while those that only remove elements from a list are implemented using simple instructions. The `list_splice_init_rcu()` primitive incurs not only memory-barrier overhead, but also grace-period latency, and is therefore the only blocking primitive shown in the table. 

**Quick Quiz 6** : Why doesn't `list_del_rcu()` poison both the `next` and `prev` pointers? 

The second pair of categories operate on the Linux kernel's `struct hlist_head`, which is a linear linked list. One advantage of `struct hlist_head` over `struct list_head` is that the former requires only a single-pointer list header, which can save significant memory in large hash tables. The `struct hlist_head` primitives in the table relate to their non-RCU counterparts in much the same way as do the `struct list_head` primitives. Their overheads are similar to that of their list counterparts in the first two categories in the table. 

The third and final pair of categories operate on Linux-kernel hlist-nulls lists, which are made up of `hlist_nulls_head` and `hlist_nulls_node` structures. These lists have special multi-valued `NULL` pointers, which have the low-order bit set to 1 with the upper bits available to the programmer to distinguish different lists. There are hlist-nulls interfaces for non-RCU-protected lists as well. 

**Quick Quiz 7** : Why would anyone need to distinguish lists based on their NULL pointers? Why not just remember which list you started searching??? 

A major advantage of hlist-nulls lists is that updaters can free elements to `SLAB_DESTROY_BY_RCU` slab caches without waiting for an RCU grace period to elapse. However, readers must be extremely careful when traversing such lists: Not only must they conduct their searches within a single RCU read-side critical section, but because any element might be freed and then reallocated at any time, readers must also validate each element that they encounter during their traversal. 

**Quick Quiz 8** : Why is there no `hlist_nulls_add_tail_rcu()`? 

####  RCU has Pointer-Based Publish-Subscribe and Version-Maintenance APIs

Although RCU's list-based APIs are quite useful, there are times when a hand-crafted data structure is required, for example, RCU-protected arrays and trees. RCU provides for these situations with the pointer-access and -update APIs in the following table: 

Category | Primitives | Purpose  
---|---|---  
Pointer update | `rcu_assign_pointer()` | Assign to an RCU-protected pointer.  
Pointer access | `rcu_dereference()` | Fetch an RCU-protected pointer, giving an lockdep-RCU error message if not in an RCU read-side critical section.  
`rcu_dereference_bh()` | Fetch an RCU-protected pointer, giving an lockdep-RCU error message if not in an RCU-bh read-side critical section.  
`rcu_dereference_sched()` | Fetch an RCU-protected pointer, giving an lockdep-RCU error message if not in an RCU-sched read-side critical section.  
`srcu_dereference()` | Fetch an RCU-protected pointer, giving an lockdep-RCU error message if not in the specified SRCU read-side critical section.  
`rcu_dereference_protected()` | Fetch an RCU-protected pointer with no protection against concurrent updates, giving an lockdep-RCU error message if the specified lockdep condition does not hold. This primitive is normally used when the update-side lock is held.  
`rcu_dereference_check()` | Fetch an RCU-protected pointer, giving an lockdep-RCU error message if (1) the specified lockdep condition does not hold and (2) not under the protection of `rcu_read_lock()`.  
`rcu_dereference_bh_check()` | Fetch an RCU-bh-protected pointer, giving an lockdep-RCU error message if (1) the specified lockdep condition does not hold and (2) not under the protection of `rcu_read_lock_bh()` (2.6.37 or later).  
`rcu_dereference_sched_check()` | Fetch an RCU-sched-protected pointer, giving an lockdep-RCU error message if (1) the specified lockdep condition does not hold and (2) not under the protection of `rcu_read_lock_sched()` or friend (2.6.37 or later).  
`srcu_dereference_check()` | Fetch an SRCU-protected pointer, giving an lockdep-RCU error message if (1) the specified lockdep condition does not hold and (2) not under the protection of the specified `srcu_read_lock()` (2.6.37 or later).  
`rcu_dereference_index_check()` | Fetch an RCU-protected integral index, giving an lockdep-RCU error message if the specified lockdep condition does not hold.  
`rcu_access_pointer()` | Fetch an RCU-protected value (pointer or index), but with no protection against concurrent updates. This primitive is normally used to do pointer comparisons, for example, to check for a `NULL` pointer.  
`rcu_dereference_raw()` | Fetch an RCU-protected pointer with no lockdep-RCU checks. Use of this primitive is strongly discouraged. If you must use this primitive, add a comment stating why, just as you would with `smp_mb()`.  
  
The `rcu_assign_pointer()` primitive assigns a new value to an RCU-protected pointer, but ensuring that any prior initialization of the pointed-to structure remains ordered before the assignment to the pointer, even on weakly ordered machines. You can think of `rcu_assign_pointer()` as publishing a new data structure to RCU readers. 

Similarly, the `rcu_dereference()` primitive reads from an RCU-protected pointer, but ensuring that subsequent code dereferencing that pointer will see the effects of initialization code prior to the corresponding `rcu_assign_pointer()`, prohibiting aggressively optimizing compilers (and the Alpha CPU) from reordering the dereferencing in a way that causes it to precede the `rcu_dereference()`. In addition, `rcu_dereference()` documents which pointer dereferences are protected by RCU. You can think of `rcu_dereference()` and similar primitives as subscribing to the current version of an RCU-protected pointer. 

**Quick Quiz 9** : Normally, any pointer subject to `rcu_dereference()` should always be updated using `rcu_assign_pointer()`. What is an exception to this rule? 

Recent changes introduced the `CONFIG_PROVE_RCU` kernel configuration parameter, which causes `rcu_dereference()` to verify that it is being used under the protection of `rcu_read_lock()`, emitting a lockdep-RCU error message (also called "lockdep-RCU splat") if not. Of course, there are multiple flavors of RCU, so that the Linux kernel now has corresponding flavors of `rcu_dereference()`, adding `rcu_dereference_bh()`, `rcu_dereference_sched()`, and `srcu_dereference()`. Use of these primitives allows you to make your code automatically check for proper use of RCU. 

It is of course also legal to access RCU-protected pointers while holding the update-side lock, in which case the data structure cannot change, which means that the compiler constraints and (on Alpha) memory barriers can be omitted. The `rcu_dereference_protected()` primitive is designed for this situation. It takes the RCU-protected pointer as its first argument and a lockdep expression as its second argument, allowing the code to verify that the required locks really are held. 

It is also legal to access RCU-protected pointers in functions that are invoked by both readers and updaters. In this case, protection against concurrent updates is still required, but the lockdep checks must allow for the possibility of RCU read-side critical sections as well as lock-based critical sections. The `rcu_dereference_check()` primitive is the tool for this job. It correctly handles concurrent updates, but also provides a second argument for a lockdep expression for update-side locks. If `rcu_dereference_check()` is called outside of the protection of `rcu_read_lock()` and of the specified locks, it will emit a lockdep-RCU splat. There are also `rcu_dereference_bh_check()`, `rcu_dereference_sched_check()`, and `srcu_dereference_check()` for RCU-bh, RCU-sched, and SRCU, respectively. 

One of the side-effects of Arnd's sparse-based checking for misuse of RCU-protected pointers is that all of the members of the `rcu_dereference()` family now require a real pointer of a type that really can be dereferenced. Further, this pointer must be a C lvalue rather than an rvalue, so that `rcu_dereference(p)` is legal, but `rcu_dereference(i)` (where `i` is an integer) and `rcu_dereference(p+1)` are not. Because there really is code in the Linux kernel with RCU-protected indexes (as opposed to pointers), the `rcu_dereference_index_check()` primitive handles the index case, and takes a lockdep expression identifying the locks and types of RCU that protect the access (see table below). Unfortunately, this also disables sparse-based checking, so it is possible that this primitive will be deprecated in the future in favor of pointers. (So if you know of an RCU-protected index that cannot be easily converted to an RCU-protected pointer, this would be a really good time to speak up!) 

In some cases, only the value of the RCU-protected pointer is used without being dereferenced; for example, the RCU-protected pointer might simply be compared against `NULL`. There is no need to protect against concurrent updates, and there is also no need to be under the protection of `rcu_read_lock()` or friends. The `rcu_access_pointer()` primitive is designed for this situation. 

Finally, RCU is used by some data structures such as radix trees, where any flavor of RCU might be used for read-side protection, or where locking might provide protection, depending on the usage. The `rcu_dereference_raw()` primitive is designed for this purpose. Note however, that this disables all lockdep-RCU checking, so please avoid using this where possible. Where it must be used, include a comment saying why its use is safe. 

#### RCU has Debugging APIs

RCU has also gained a number of debugging APIs, shown in the following table: 

Category | Primitives | Purpose  
---|---|---  
RCU pointer declaration | `__rcu` | Tell sparse that a given pointer is intended to be protected by RCU.  
Callback tracking | `init_rcu_head_on_stack()` | Prepare on-stack callback for debug-objects (`CONFIG_DEBUG_OBJECTS_RCU_HEAD`).  
`destroy_rcu_head_on_stack()` | Prepare on-stack callback for exiting scope.  
Critical section validation | `rcu_read_lock_held()` | Splat unless under `rcu_read_lock()`.  
`rcu_read_lock_bh_held()` | Splat unless under `rcu_read_lock_bh()` or unless interrupts are disabled.  
`rcu_read_lock_sched_held()` | Splat unless under `rcu_read_lock_sched()` or friend.  
`srcu_read_lock_held()` | Splat unless under specified `srcu_read_lock()`.  
  
The first debugging API is used to mark an RCU-protected pointer, for example: 

```
struct foo __rcu *foo_p;
```

If a pointer is marked with `__rcu`, then all accesses to this pointer must be made via `rcu_assign_pointer()` or via one of the `rcu_dereference()` family of APIs, otherwise sparse will flag the error when the `CONFIG_SPARSE_RCU_POINTER` kernel configuration parameter is set. Note that `__rcu` may be used to tag arguments passed in to functions as well as global variables, struct fields, and the like. 

Just as a double `kfree()` is a bug, so is a double invocation of `call_rcu()` using the same pointer. If the `CONFIG_DEBUG_OBJECTS_RCU_HEAD` kernel configuration parameter is set, then debug objects will flag this sort of abuse. However, the debug-objects mechanism does not track objects on the stack by default. Therefore, when working with a stack-based `rcu_head` structure, use `init_rcu_head_on_stack()` before the first use and `destroy_rcu_head_on_stack()` before the corresponding stack frame is deallocated. 

Finally, it is common practice to include a header comment on functions that require RCU protection. Although this is good practice, it is far better to enlist the computer's help in locating cases where RCU protection has been omitted. The `rcu_read_lock_held()`, `rcu_read_lock_bh_held()`, `rcu_read_lock_sched_held()`, and `srcu_read_lock_held()` primitives enable debug checks in the code itself. These primitives provide the most useful information when the `CONFIG_PROVE_RCU` kernel parameter is set. 

These debugging APIs can be extremely helpful. It does take a little extra work to get them set up correctly, but much less work than is required to debug RCU usage problems the hard way!!! 

**Quick Quiz 10** : Why isn't there a debugging API to check for mismatching read-side and grace-period primitives, for example, using `rcu_read_lock()` on the read side, but incorrectly using `synchronize_sched()` during updates? 

####  Kernel Configuration Parameters

RCU's kernel-configuration parameters can be considered to be part of the RCU API, most especially from the viewpoint of someone building a kernel intended for a specialized device or workload. This section summarizes the RCU-related configuration parameters. 

The first set of parameters controls the underlying behavior of the RCU implementation itself, and is defined in `init/Kconfig`. 

  1. `CONFIG_TREE_RCU`: selects the non-preemptible tree-based RCU implementation that is appropriate for server-class SMP builds. It can accommodate a very large number of CPUs, but scales down sufficiently well for all but the most memory-constrained systems. The following kernel parameters may also be specified when `CONFIG_TREE_RCU` is specified: 
     1. `CONFIG_RCU_TRACE` enables `debugfs`-based tracing. 
     2. `CONFIG_RCU_FANOUT` controls the fanout of the tree. Lower fanout values reduce lock contention, but also consumes more memory and increases the overhead of grace-period computations. To the best of my knowledge, the default values have been always been sufficient. 
     3. `CONFIG_RCU_FANOUT_EXACT` forces the tree to be as balanced as possible. Again, to the best of my knowledge, the default values have always been sufficient. 
     4. `CONFIG_RCU_FAST_NO_HZ` causes the last non-dyntick-idle CPU to be more aggressive about entering dyntick-idle state. If you are not working with an SMP battery-powered device, you probably don't care about this parameter.  In addition, the following parameters may be specified, either on the boot command line or via `sysfs`: 
     1. `blimit`: This specifies the maximum number of RCU callbacks that may be processed consecutively, and defaults to 10. 
     2. `qhimark`: If the number of callbacks waiting on a given CPU exceeds this number, which defaults to 10,000, then `blimit` is ignored and RCU callbacks will be processed indefinitely if need be. 
     3. `qlomark`: If a given CPU is in indefinite-callback mode, then it will return to normal `blimit`-based callback processing once the number of outstanding RCU callbacks drops below `qlomark`, which defaults to 100. 

  2. `CONFIG_TREE_PREEMPT_RCU`: selects the preemptible tree-based RCU implementation that is appropriate for real-time and low-latency SMP builds. It can also accommodate a very large number of CPUs, and also scales down sufficiently well for all but the most memory-constrained systems. The `CONFIG_RCU_TRACE`, `CONFIG_RCU_FANOUT`, `CONFIG_RCU_FANOUT_EXACT`, `blimit`, `qhimark`, and `qlomark` parameters are supported as noted above. 

  3. `CONFIG_TINY_RCU`: selects the non-preemptible uniprocessor RCU implementation that is appropriate for non-real-time UP builds. It has the smallest memory footprint of any of the current in-kernel RCU implementations. 

  4. `CONFIG_TINY_PREEMPT_RCU`: selects the preemptible uniprocessor RCU implementation that is appropriate for real-time UP builds. It also boasts a small memory footprint, though not quite so small as `CONFIG_TINY_RCU`. This implementation was produced as part of the [Linaro](<http://www.linaro.org/>) effort. 

The second set of kernel configuration parameters controls debugging options: 

  1. `CONFIG_SPARSE_RCU_POINTER` enables sparse-based checks of proper use of RCU-protected pointers. Please note that this is a build-time check: Use "make C=1" to cause sparse to check source files that would have been rebuilt by "make", and use "make C=2" to cause sparse to unconditionally check source files. 

  2. `CONFIG_DEBUG_OBJECTS_RCU_HEAD` enables debug-objects checking of multiple invocations of `call_rcu` (and friends) on the same structure. 

  3. `CONFIG_PROVE_RCU` enables lockdep-RCU checking. If `CONFIG_PROVE_RCU_REPEATEDLY` is also specified, then the lockdep-RCU checking can output multiple lockdep-RCU "splats", otherwise only a single lockdep-RCU splat will be emitted per boot. 

  4. `CONFIG_RCU_TORTURE_TEST` enables RCU torture testing. This is a tri-state parameter, permitting `rcutorture.c` to be compiled into the kernel, built as a module, or omitted entirely. When `rcutorture.c` is built into the kernel (`CONFIG_RCU_TORTURE_TEST=y`), then `CONFIG_RCU_TORTURE_TEST_RUNNABLE` starts RCU torture testing during boot. Please don't try this on a production system! 

  5. `CONFIG_RCU_CPU_STALL_DETECTOR`, which is enabled by default for `CONFIG_TREE_RCU` and `CONFIG_TREE_PREEMPT_RCU` kernels, provides RCU-based checking for CPU stalls, which occur when a CPU or task fails to find its way out of an RCU read-side critical section in a timely manner. CPU stalls can be caused by a number of bugs, as described in `Documentation/RCU/stallwarn.txt`. The stall-warning timeout is controlled by `CONFIG_RCU_CPU_STALL_TIMEOUT`, which defaults to 60 seconds. If `CONFIG_RCU_CPU_STALL_DETECTOR_RUNNABLE` is set, which it is by default, then CPU stalls are checked for starting early in boot, otherwise, the `rcu_cpu_stall_suppress` module parameter must be manually specified (via either the boot-time command line or via `sysfs`) in order to start CPU stall checking. If `CONFIG_RCU_CPU_STALL_VERBOSE` is set, which it also is by default, then detailed per-task information is printed when a CPU stall is encountered. 

If you are working with code that uses RCU, please do us all a favor and test that code with `CONFIG_PROVE_RCU` and `CONFIG_DEBUG_OBJECTS_RCU_HEAD` enabled. Please also consider running sparse with `CONFIG_SPARSE_RCU_POINTER`. If you are modifying the RCU implementation itself, you will need to run rcutorture, with multiple runs covering the relevant kernel configuration parameters. A one-hour rcutorture run on an 8-CPU machine qualifies as light rcutorture testing. 

Yes, running extra tests can be a hassle, but I am here to tell you that extra testing is _much_ easier than trying to track down bugs in your RCU code!!! 

####  What Next for the RCU API?

The most honest answer is that I do not know. The next steps for the RCU API will be decided as they always have been, by the needs of RCU's users and by the limits of the technology at the time. That said, the following seem to be a few of the more likely directions: 

  1. Complete implementation of RCU priority boosting (`TINY_RCU` submission slated for 2.6.38, `TREE_RCU` implementation in progress). 

  2. Support for running certain types of user applications without scheduler ticks. There was a spirited discussion on this topic at Linux Plumbers Conference, but at this writing it appears that only small tweaks to RCU will be required. 

  3. Merge the implementation of SRCU into `TINY_RCU` and `TREE_RCU`. A design for this is in mostly in place. This effort is likely to result in `call_srcu()` and `srcu_barrier()`. If it does, please be _very_ careful with these primitives!!! 

  4. Make `RCU_FAST_NO_HZ` work for `TREE_PREEMPT_RCU`. 

  5. Drive the choice between `TINY_PREEMPT_RCU`, `TINY_RCU`, `TREE_PREEMPT_RCU`, and `TREE_RCU` entirely off of `SMP` and `PREEMPT`. This would allow cutting code and test scenarios, but first `TINY_PREEMPT_RCU` must prove itself. 

  6. It is possible that `rcu_dereference_index_check()` will be retired if it is reasonable to convert all current use of RCU-protected indexes into RCU-protected pointers. 

  7. It is quite possible that large systems might encounter problems with `synchronize_rcu_expedited()` scalability. 

  8. Make RCU be more aggressive about entering dyntick-idle state when running on lightly loaded systems with four or more CPUs. 

  9. Numerous other items listed on the [RCU to-do list](<http://kernel.org/pub/linux/kernel/people/paulmck/rcutodo.html>). 

But if the past is any guide, new use cases and workloads will place unanticipated demands on RCU. 

#### Acknowledgments

We are all indebted to Andy Whitcroft, Jon Walpole, Gautham Shenoy, and LWN member jarkao2, whose review of early drafts of this document greatly improved it. A great many people helped find and address issues located by the software-engineering enhancements to RCU, including Andi Kleen, Andrew Morton, Avi Kivity, Ben Greear, Daniel J Blueman, David Howells, Dhaval Giani, Eric Dumazet, Eric Paris, Frederic Weisbecker, Greg Thelen, Heiko Carstens, Ilia Mirkin, Ingo Molnar, Jens Axboe, Jiri Slaby, Johannes Berg, KAMEZAWA Hiroyuki, KOSAKI Motohiro, Lai Jiangshan, Marcelo Tosatti, Mathieu Desnoyers, Miles Lane, Minchan Kim, Oleg Nesterov, Paul Moore, Peter Zijlstra, Robert Olsson, Sergey Senozhatsky, Subrata Modak, Tetsuo Handa, Thomas Gleixner, Trond Myklebust, Valdis Kletnieks, Vegard Nossum, Vivek Goyal, and Zdenek Kabelac. I owe thanks to the members of the Relativistic Programming project and to members of PNW TEC for many valuable discussions. I am grateful to Dan Frye for his support of this effort. 

This work represents the view of the author and does not necessarily represent the view of IBM. 

Linux is a registered trademark of Linus Torvalds. 

Other company, product, and service names may be trademarks or service marks of others. 

####  Answers to Quick Quizzes

**Quick Quiz 1** : Why are some of the cells in the above table colored green? 

**Answer** : The green API members (`rcu_read_lock()`, `rcu_read_unlock()`, and `call_rcu()`) were the only members of the RCU (then called "rclock") API back in the mid-90s. During this timeframe, Paul was under the mistaken impression that he knew all that there is to know about RCU. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Why are some of the cells in the above table colored blue? 

**Answer** : Because the corresponding API members are new since the 2008 version of [RCU part 3: the RCU API](<http://lwn.net/Articles/264090/>). 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : What happens if you mix and match RCU and RCU Sched? 

**Answer** : In a `CONFIG_TREE_RCU` or a `CONFIG_TINY_RCU` kernel, mixing these two works "by accident" because in those kernel builds, RCU and RCU Sched map to the same implementation. However, this mixture is fatal in `CONFIG_TREE_PREEMPT_RCU` and `CONFIG_TINY_PREEMPT_RCU` builds, due to the fact that RCU's read-side critical sections can then be preempted, which would permit `synchronize_sched()` to return before the RCU read-side critical section reached its `rcu_read_unlock()` call. This could in turn result in a data structure being freed before the read-side critical section was finished with it, which could in turn greatly increase the actuarial risk experienced by your kernel. 

Even in `CONFIG_TREE_RCU` and `CONFIG_TINY_RCU` builds, such mixing and matching is of course very strongly discouraged. Mixing and matching other flavors of RCU is also a very bad idea. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : Why does SRCU lack an asynchronous `call_srcu()` interface? 

**Answer** : Given an asynchronous interface, a single task could register an arbitrarily large number of SRCU callbacks, thereby consuming an arbitrarily large quantity of memory. In contrast, given the current synchronous `synchronize_srcu()` interface, a given task must finish waiting for a given grace period before it can start waiting for the next one. 

However, there is a good chance that a `call_srcu()` will become available in the near future. If it does, and if you decide to use it, please be careful!!! 

**Back to Quick Quiz 4**.

**Quick Quiz 5** : Can `synchronize_srcu()` be safely used within an SRCU read-side critical section? If so, why? If not, why not? 

**Answer** : In principle, you can use `synchronize_srcu()` with a given `srcu_struct` within an SRCU read-side critical section that uses some other `srcu_struct`. In practice, however, such use is almost certainly a bad idea. In particular, the following could still result in deadlock: 

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

**Back to Quick Quiz 5**.

**Quick Quiz 6** : Why doesn't `list_del_rcu()` poison both the `next` and `prev` pointers? 

**Answer** : Poisoning the `next` pointer would interfere with concurrent RCU readers, which must use this pointer. However, RCU readers are forbidden from using the `prev` pointer, so it may safely be poisoned. 

**Back to Quick Quiz 6**.

**Quick Quiz 7** : Why would anyone need to distinguish lists based on their NULL pointers? Why not just remember which list you started searching??? 

**Answer** : Suppose that CPU 0 is traversing such a list within an RCU read-side critical section, where the elements are allocated from `SLAB_DESTROY_BY_RCU` slab cache. The elements could therefore be freed and reallocated at any time. If CPU 0 is referencing an element while CPU 1 is freeing that element, and if CPU 1 then quickly reallocates that same element and adds it to some other list, then CPU 0 will be transported to that new list along with the element. In this case, remembering the starting list would clearly be unhelpful. 

To make matters worse, suppose that CPU 0 searches a list and fails to find the element that it was looking for. Was that because the element did not exist? Or because CPU 0 got transported to some other list in the meantime? Readers traversing `SLAB_DESTROY_BY_RCU` lists must carefully validate each element and check for being moved to another list. One way to check for being moved to another list is for each list to have its own value for the `NULL` pointer. These checks are subtle and easy to get wrong, so please be careful! 

**Back to Quick Quiz 7**.

**Quick Quiz 8** : Why is there no `hlist_nulls_add_tail_rcu()`? 

**Answer** : Suppose that CPU 0 is traversing an hlist-nulls list under RCU protection. Suppose that while CPU 0 is referencing list element A, CPU 1 frees it and reallocates it, adding it to another list, which CPU 0 unwittingly starts traversing. Suppose further that while CPU 0 is referencing an element B in the new list, CPU 2 frees and reallocates it, moving it back to the original list. When CPU 0 comes to the end of the original list, it sees that the `NULL` pointer has the proper value, so does not realize that it has been moved. 

If CPU 2 had added element B at the tail of the list, CPU 0 would be within its rights to conclude that it had fully searched this list when in fact it had not. But given that it is only possible to add elements to the head of an hlist-nulls list, any CPU coming to the end of the same list it started traversing can be sure that it really did search the entire list. Possibly several times, if it was extremely unlucky. 

Therefore, there is not and should never be a primitive to add to the middle or the end of an RCU-protected hlist-nulls list. Except maybe at initialization time, before any readers have access to the list. 

**Back to Quick Quiz 8**.

**Quick Quiz 9** : Normally, any pointer subject to `rcu_dereference()` _must_ always be updated using `rcu_assign_pointer()`. What is an exception to this rule? 

**Answer** : One such exception is when a multi-element linked data structure is initialized as a unit while inaccessible to other CPUs, and then a single `rcu_assign_pointer()` is used to plant a global pointer to this data structure. The initialization-time pointer assignments need not use `rcu_assign_pointer()`, though any such assignments that happen after the structure is globally visible _must_ use `rcu_assign_pointer()`. 

However, unless this initialization code is on an impressively hot code-path, it is probably wise to use `rcu_assign_pointer()` anyway, even though it is in theory unnecessary. It is all too easy for a "minor" change to invalidate your cherished assumptions about the initialization happening privately. Besides, Arnd's sparse-based checks will yell at you if you apply simple assignment to a pointer that has been marked as protected by RCU using `__rcu`. 

**Back to Quick Quiz 9**.

**Quick Quiz 10** : Why isn't there a debugging API to check for mismatching read-side and grace-period primitives, for example, using `rcu_read_lock()` on the read side, but incorrectly using `synchronize_sched()` during updates? 

**Answer** : Because we have not yet come up with a good way to do this. One challenge is that `synchronize_sched()` doesn't say which RCU pointer it is waiting on. Even if the pointer was known, the connection to the read-side primitives might well be in some other compilation unit. One approach would be to use separate pointer tags for each flavor of RCU rather than the current `__rcu`, but early attempts in that direction resulted in mind-numbing complexity -- even from an RCU perspective. 

If you have a good idea, give it a try and let us know how it goes! 

**Back to Quick Quiz 10**.
