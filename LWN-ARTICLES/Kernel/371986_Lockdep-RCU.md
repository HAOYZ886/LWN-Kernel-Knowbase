---
title: "Lockdep-RCU"
url: https://lwn.net/Articles/371986/
date: "February 1, 2010"
category: "Read-copy-update"
author: "February 1, 2010 This article was contributed by Paul McKenney"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

February 1, 2010

This article was contributed by Paul McKenney

#### Introduction

Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October of 2002. RCU improves scalability by allowing readers to execute concurrently with writers. In contrast, conventional locking primitives require that readers wait for ongoing writers and vice versa. RCU ensures read coherence by maintaining multiple versions of data structures and ensuring that they are not freed until all pre-existing read-side critical sections complete. RCU relies on efficient and scalable mechanisms for publishing and reading new versions of an object, and also for deferring the collection of old versions. These mechanisms distribute the work among read and update paths in such a way as to make read paths extremely fast. In some cases (non-preemptable kernels), RCU's read-side primitives have zero overhead. RCU updates can be expensive, so RCU is in general best-suited to read-mostly data structures. 

RCU readers execute in _RCU read-side critical sections_ that begin with `rcu_read_lock()` and end with `rcu_read_unlock()`. The Linux kernel has [multiple flavors of RCU](<http://lwn.net/Articles/264090/>), and each flavor uses its own flavor of `rcu_read_lock()` and `rcu_read_unlock()`. Anything outside of an RCU read-side critical section is a _quiescent state_ , and a _grace period_ is any time period in which every CPU (or task, for real-time RCU implementations) passes through at least one quiescent state. Taken together, these rules guarantee that any RCU read-side critical section that is executing at the beginning of a given grace period must complete before that grace period can be permitted to end. 

This guarantee is surprisingly useful, allowing RCU to act as a high-performance scalable replacement for reader-writer locking, [among other things](<http://lwn.net/Articles/263130/>). But this guarantee is sufficient only for systems with sequentially consistent memory ordering, which are quite rare. Even strongly ordered architectures such as x86 or s390 will allow later reads to execute ahead of prior writes, and compilers can reorder code quite freely. Therefore, RCU needs an additional [publish-subscribe](<http://lwn.net/Articles/262464/>) guarantee, which is provided by `rcu_assign_pointer()` and `rcu_dereference()`. Uses of `rcu_assign_pointer()` are typically protected by the update-side lock, and uses of `rcu_dereference()` must typically be within an RCU-read-side critical section. 

Unfortunately for this simple rule on use of `rcu_dereference()`, there is quite a bit of code that is used by both RCU readers and updaters. A more accurate rule is that `rcu_dereference()` must either be: 

  1. within an RCU read-side critical section, 
  2. protected by the update-side lock, or 
  3. inaccessible to RCU readers. 

The remainder of this article is as follows: 

  1. Why Bother With lockdep-Enabling RCU?
  2. RCU API for lockdep.
  3. RCU lockdep Usage Examples.
  4. RCU lockdep Implementation.
  5. RCU API for lockdep: Quick Reference. 
These sections are followed by  Conclusions and Future Directions and  Answers to Quick Quizzes. 

####  Why Bother With lockdep-Enabling RCU?

Compliance with the usage rule for `rcu_dereference()` is verified by manual code inspection. And this manual code inspection worked great back in 2.6.10, when there were at grand total of 38 occurrences of `rcu_dereference()`. However, given that there are now more than 350 occurrences of `rcu_dereference()` in 2.6.32, it appears the day of sole reliance on manual code inspection is long over. Additional evidence on this point was provided by Thomas Gleixner when he [trained his eagle eye](<http://lwn.net/Articles/366717/>) on a few `rcu_dereference()` instances in mainline. 

It is clearly time to bring lockdep-style checking to `rcu_dereference()`. Unfortunately, because `rcu_dereference_check()` can be used in such a wide variety of environments, simple addition of lockdep checking to the current API fails, producing reams of false positives while ignoring potentially dangerous bugs. 

**Quick Quiz 1** : How can you be so sure that there is no clever lockdep-check strategy given the current API? Answer

####  RCU API for lockdep

Some major goals of any API change is to minimize impact on existing code, patches in flight, and ongoing debugging efforts. 

Because the most common use of `rcu_dereference()` is for accesses that are strictly within a vanilla RCU read-side critical section, `rcu_dereference()` should check only for being in a vanilla RCU read-side critical section. This minimizes impact on existing code, including patches in flight. This means that other `rcu_dereference()` API members must be created. 

However, these other API members cannot be defined in terms of `rcu_dereference()` because these other members must be usable outside of vanilla RCU read-side critical sections. Therefore, a raw interface named `rcu_dereference_raw()` inherits the implementation that used to belong to `rcu_dereference()`. In other words, if you "know what you are doing", just use `rcu_dereference_raw()` and lockdep will never complain about them. (But you just might hear a few questions from me!) 

The underlying API for the other forms of `rcu_dereference()` is `rcu_dereference_check()`, which takes two arguments. The first argument is an RCU-protected pointer, the same as that of `rcu_dereference()` and the new `rcu_dereference_raw()`. The second argument is a boolean expression that evaluates to zero if there is a problem, in which case, if RCU lockdep is enabled, you will get a `WARN_ON_ONCE()` on your console log. 

The other dereferencing APIs are `rcu_dereference()`, `rcu_dereference_sched()`, `rcu_dereference_bh()`, and `srcu_dereference()`, each of which checks to make sure that it is being used in the corresponding flavor of RCU read-side critical section, giving your console log a `WARN_ON_ONCE()` otherwise (again, assuming that RCU lockdep is enabled). All of these take a single RCU-protected pointer as an argument, except for `srcu_dereference()`, which also takes a pointer to a `struct srcu_struct`. This additional argument permits `srcu_dereference()` to distinguish among multiple SRCU domains. 

These four dereferencing APIs use corresponding APIs that check for being in the corresponding flavor of RCU read-side critical section: `rcu_read_lock_held()`, `rcu_read_lock_bh_held()`, `rcu_read_lock_sched_held()`, and `srcu_read_lock_held()`. Of these, only `srcu_read_lock_held()` takes an argument, namely a `struct srcu_struct`, again permitting distinguishing among multiple SRCU domains. 

####  RCU lockdep Usage Examples

The prototypical use of these new APIs is as follows: 

```
1 rcu_read_lock();
      2 p = rcu_dereference(gp->data);
      3 do_something_with(p);
      4 rcu_read_unlock();
```

The alert reader may have noticed that this is no different from the old usage of these APIs. This situation is strictly intentional. 

Similar code may be written for other flavors of RCU, for example: 

```
1 srcu_read_lock();
      2 p = srcu_dereference(gp->data, sp);
      3 do_something_with(p);
      4 srcu_read_unlock();
```

These examples work well when used inside RCU read-side critical sections, but fail completely for code that is invoked both by readers and updaters. Although we could insert artificial RCU read-side critical sections in updaters, these can cause much confusion. Instead, we use `rcu_dereference_check()`, for example, in the `files_fdtable()` macro: 

```
1 #define files_fdtable(files) \
      2   (rcu_dereference_check((files)->fdt, \
      3                          rcu_read_lock_held() || \
      4                          lockdep_is_held(&(files)->file_lock) || \
      5                          atomic_read(&files->count) == 1))
```

This statement fetches the RCU-protected pointer `(files)->fdt`, but requires that `files_fdtable()` be invoked within an RCU read-side critical section, with `lockdep_is_held(&(files)->file_lock)` held, or with the `&files->count` reference counter zeroed (in other words, if inaccessible to RCU readers). 

**Quick Quiz 2** : Suppose that an access to an RCU-protected pointer `gp` must be either inside an RCU-bh read-side critical section, an SRCU read-side critical section for SRCU domain `sp`, or with `mylock` held. How do you code this? Answer

####  RCU lockdep Implementation

The basic change underlying the RCU lockdep implementation is a set of per-RCU-flavor lockdep maps (in the case of SRCU, per-SRCU-domains lockdep maps `->depmap` in each `struct srcu_struct`): 

```
1 extern struct lockdep_map rcu_lock_map;
      2 # define rcu_read_acquire() \
      3     lock_acquire(&rcu_lock_map, 0, 0, 2, 1, NULL, _THIS_IP_)
      4 # define rcu_read_release()  lock_release(&rcu_lock_map, 1, _THIS_IP_)
      5 
      6 extern struct lockdep_map rcu_bh_lock_map;
      7 # define rcu_read_acquire_bh() \
      8     lock_acquire(&rcu_bh_lock_map, 0, 0, 2, 1, NULL, _THIS_IP_)
      9 # define rcu_read_release_bh()  lock_release(&rcu_bh_lock_map, 1, _THIS_IP_)
     10 
     11 extern struct lockdep_map rcu_sched_lock_map;
     12 # define rcu_read_acquire_sched() \
     13     lock_acquire(&rcu_sched_lock_map, 0, 0, 2, 1, NULL, _THIS_IP_)
     14 # define rcu_read_release_sched() \
     15     lock_release(&rcu_sched_lock_map, 1, _THIS_IP_)
     16 
     17 # define srcu_read_acquire(sp) \
     18     lock_acquire(&(sp)->dep_map, 0, 0, 2, 1, NULL, _THIS_IP_)
     19 # define srcu_read_release(sp) \
     20     lock_release(&(sp)->dep_map, 1, _THIS_IP_)
```

These are used to implement `rcu_read_lock_held()`, `rcu_read_lock_bh_held()`, `rcu_read_lock_sched_held()`, and `srcu_read_lock_held()`: 

```
1 static inline int rcu_read_lock_held(void)
      2 {
      3   if (debug_locks)
      4     return lock_is_held(&rcu_lock_map);
      5   return 1;
      6 }
      7 
      8 static inline int rcu_read_lock_bh_held(void)
      9 {
     10   if (debug_locks)
     11     return lock_is_held(&rcu_bh_lock_map);
     12   return 1;
     13 }
     14 
     15 static inline int rcu_read_lock_sched_held(void)
     16 {
     17   int lockdep_opinion = 0;
     18 
     19   if (debug_locks)
     20     lockdep_opinion = lock_is_held(&rcu_sched_lock_map);
     21   return lockdep_opinion || preempt_count() != 0;
     22 }
     23 
     24 static inline int srcu_read_lock_held(struct srcu_struct *sp)
     25 {
     26   if (debug_locks)
     27     return lock_is_held(&sp->dep_map);
     28   return 1;
     29 }
```

In each case, if lockdep is enabled, we consult the corresponding `lockdep_map`, otherwise, we (conservatively) guess that we are in the appropriate RCU read-side critical section. This permits `WARN_ON_ONCE(!rcu_read_lock_held())` to be used freely. 

**Quick Quiz 3** : How do these work if lockdep is not configured at all? Answer

The non-checking variant of `rcu_dereference()` is `rcu_dereference_raw()`, which is defined as follows: 

```
1 #define rcu_dereference_raw(p)  ({ \
      2         typeof(p) _________p1 = ACCESS_ONCE(p); \
      3         smp_read_barrier_depends(); \
      4         (_________p1); \
      5         })
```

Then `rcu_dereference_check()` is implemented in terms of `rcu_dereference_raw()` as follows: 

```
1 #define rcu_dereference_check(p, c) \
      2   ({ \
      3     if (debug_locks) \
      4       WARN_ON_ONCE(!(c)); \
      5     rcu_dereference_raw(p); \
      6   })
```

However, if lockdep is not configured, the following alternative implementation is used: 

```
1 #define rcu_dereference_check(p, c)     rcu_dereference_raw(p)
```

**Quick Quiz 4** : Why not include a `((void)(c))` to the non-lockdep version of `rcu_dereference_check()` in order to detect compiler errors in the "c" argument? Answer

The remainder of the primitives are defined as follows: 

```
1 #define rcu_dereference(p) \
      2   rcu_dereference_check(p, rcu_read_lock_held())
      3 
      4 #define rcu_dereference_bh(p) \
      5     rcu_dereference_check(p, rcu_read_lock_bh_held())
      6 
      7 #define rcu_dereference_sched(p) \
      8     rcu_dereference_check(p, rcu_read_lock_sched_held())
      9 
     10 #define srcu_dereference(p, sp) \
     11     rcu_dereference_check(p, srcu_read_lock_held(sp))
```

**Quick Quiz 5** : What are the non-lockdep definitions of these primitives? Answer

####  RCU API for lockdep: Quick Reference

> Name | CONFIG_PROVE_RCU | !CONFIG_PROVE_RCU  
> ---|---|---  
> rcu_dereference(p) | returns p, warns if not in RCU read-side critical section | returns p, never warns  
> rcu_dereference_bh(p) | returns p, warns if not in RCU-bh read-side critical section | returns p, never warns  
> rcu_dereference_sched(p) | returns p, warns if not in RCU-sched read-side critical section | returns p, never warns  
> srcu_dereference(p, sp) | returns p, warns if not in SRCU read-side critical section for sp | returns p, never warns  
> rcu_dereference_check(p, c) | returns p, warns if !c | returns p, never warns  
> rcu_dereference_raw(p) | returns p, never warns | returns p, never warns  
>    
> rcu_read_lock_held() | non-zero if in RCU read-side critical section | always non-zero  
> rcu_read_lock_bh_held() | non-zero if in RCU-bh read-side critical section | always non-zero  
> rcu_read_lock_sched_held() | non-zero if in RCU-sched read-side critical section | always non-zero  
> srcu_read_lock_held(sp) | non-zero if in SRCU read-side critical section for sp | always non-zero  
  
####  Conclusions and Future Directions

These are early days for the lockdep-enabled RCU primitives. They have been applied to some of the networking, VFS, scheduler, radix tree, and IDR code. Thus far, things are going well, but here are some possible future directions: 

  1. The RCU list macros, radix tree, and IDR implementations currently use `rcu_dereference_raw()`. At some point, it may be necessary to produce checked variants. Given that this will require yet more APIs, need must be demonstrated before the API explosion is undertaken. `list_for_each_rcu()`, `list_for_each_rcu_bh()`, `list_for_each_rcu_sched()`, `list_for_each_srcu()`, `list_for_each_rcu_check()`, and `list_for_each_rcu_raw()`, anyone? 

  2. Thus far, it has been easy to generate `rcu_dereference_check()`'s boolean expressions. Nevertheless, I am a bit nervous about code that is called both in RCU read-side critical sections and by initialization code. In some cases, it might be difficult to detect the initialization case, but this will be dealt with as they come up. 

  3. The `rcu_assign_pointer()` primitive remains unchecked. It is used primarily under locks, which are quite a bit more familiar, and for which there is already lockdep available. 

Regardless of how the future unfolds, lockdep-enabled RCU should be very helpful in detecting RCU-usage bugs. 

#### Acknowledgments

I am grateful to Peter Zijlstra and Thomas Gleixner for sharing their experiences applying lockdep checking to `rcu_dereference()`. I owe thanks to Eric Dumazet for helping me work out how to handle some difficult `rcu_dereference()` instances in the networking code, to Ingo Molnar for much encouragement and advice, and to Kathy Bennett for her support of this effort. 

This work represents the view of the authors and does not necessarily represent the view of IBM. 

####  Answers to Quick Quizzes

**Quick Quiz 1**: How can you be so sure that there is no clever lockdep-check strategy given the current API? 

**Answer** : Because if there was a clever lockdep-check strategy given the current RCU API, Peter Zijlstra would have implemented it! If you know of one, please don't keep it a secret -- but please do yourself the favor of reading the rest of this article before deciding whether or not you do have a solution. 

**Back to Quick Quiz 1**.

**Quick Quiz 2**: Suppose that an access to an RCU-protected pointer `gp` must be either inside an RCU-bh read-side critical section, an SRCU read-side critical section for SRCU domain `sp`, or with `mylock` held. How do you code this? 

**Answer** : One approach is as follows: 

```
1   rcu_dereference_check(gp,
      2                         rcu_read_lock_bh_held() ||
      3                         srcu_read_lock_held(sp) ||
      4                         lockdep_is_held(&mylock));
```

**Back to Quick Quiz 2**.

**Quick Quiz 3**: How do these work if lockdep is not configured at all? 

**Answer** : As follows: 

```
1 static inline int rcu_read_lock_held(void)
      2 {
      3   return 1;
      4 }
      5 
      6 static inline int rcu_read_lock_bh_held(void)
      7 {
      8   return 1;
      9 }
     10 
     11 static inline int rcu_read_lock_sched_held(void)
     12 {
     13   return preempt_count() != 0;
     14 }
     15 
     16 static inline int srcu_read_lock_held(struct srcu_struct *sp)
     17 {
     18   return 1;
     19 }
```

**Back to Quick Quiz 3**.

**Quick Quiz 4**: Why not include a `((void)(c))` to the non-lockdep version of `rcu_dereference_check()` in order to detect compiler errors in the "c" argument? 

**Answer** : Because `lockdep_is_held()` is defined only in lockdep builds of the kernel. Therefore, `((void)(c))` would give you lots of false alarms. So, just make sure that you do at least one build-and-test cycle with lockdep defined. 

**Back to Quick Quiz 4**.

**Quick Quiz 5**: What are the non-lockdep definitions of these primitives? 

**Answer** : They are exactly the same as the lockdep definitions! The implementations of `rcu_dereference_check()` remove the need for duplicate definitions for `rcu_dereference()`, `rcu_dereference_bh()`, `rcu_dereference_sched()`, and `srcu_dereference()`. 

**Back to Quick Quiz 5**.
