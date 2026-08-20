---
title: "Requirements for RCU part 1: the fundamentals"
url: https://lwn.net/Articles/652156/
date: "July 29, 2015"
category: "Read-copy-update"
author: "July 29, 2015 This article was contributed by Paul McKenney"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

July 29, 2015

This article was contributed by Paul McKenney

Read-copy update (RCU) is a synchronization mechanism that is often used as a replacement for reader-writer locking. RCU is unusual in that updaters do not block readers, which means that RCU's read-side primitives can be exceedingly fast and scalable. In addition, updaters can make useful forward progress concurrently with readers. However, all this concurrency between RCU readers and updaters does raise the question of exactly what RCU readers are doing, which in turn raises the question of exactly what RCU's requirements are. 

This document, to be published in three parts, therefore summarizes RCU's requirements, and can be thought of as an informal, high-level specification for RCU. It is important to understand that RCU's specification is primarily empirical in nature; in fact, I learned about many of these requirements the hard way. This situation might cause some consternation, however, not only has this learning process been a lot of fun, but it has also been a great privilege to work with so many people willing to apply technologies in interesting new ways. 

All that aside, the following categories of known RCU requirements will be covered in this article: 

  1. Fundamental Requirements
  2. Fundamental Non-Requirements 

This is followed by, of course, the inevitable answers to the quick quizzes. The following articles will continue the exploration of this topic with several other categories of requirements. 

#### Fundamental requirements

RCU's fundamental requirements are the closest thing RCU has to hard mathematical requirements. These are: 

  1. Grace-Period Guarantee
  2. Publish-Subscribe Guarantee
  3. RCU Primitives Guaranteed to Execute Unconditionally
  4. Guaranteed Read-to-Write Upgrade 

#### Grace-period guarantee

RCU's grace-period guarantee is unusual in being premeditated: Jack Slingwine and I had this guarantee firmly in mind when we started work on RCU (then called "rclock") in the early 1990s. That said, the past two decades of experience with RCU have produced a much more detailed understanding of this guarantee. 

RCU's grace-period guarantee allows updaters to wait for all pre-existing RCU read-side critical sections to complete. An RCU read-side critical section begins with the marker `rcu_read_lock()` and ends with the marker `rcu_read_unlock()`. These markers may be nested, and RCU treats a nested set as one big RCU read-side critical section. Production-quality implementations of `rcu_read_lock()` and `rcu_read_unlock()` are extremely lightweight, and in fact have exactly zero overhead in Linux kernels built for production use with `CONFIG_PREEMPT=n`. 

This guarantee allows ordering to be enforced with extremely low overhead to readers, for example: 

> 

```
>      1 int x = 0, y = 0;
>      2 
>      3 void thread0(void)
>      4 {
>      5   rcu_read_lock();
>      6   r1 = READ_ONCE(x);
>      7   r2 = READ_ONCE(y);
>      8   rcu_read_unlock();
>      9 }
>     10 
>     11 void thread1(void)
>     12 {
>     13   WRITE_ONCE(x, 1);
>     14   synchronize_rcu();
>     15   WRITE_ONCE(y, 1);
>     16 }
>
```

**Quick Quiz 1** : Wait a minute! You said that updaters can make useful forward progress concurrently with readers, but pre-existing readers will block `synchronize_rcu()`!!! Just who are you trying to fool???   
Answer

Because the `synchronize_rcu()` on line 14 waits for all pre-existing readers, any instance of `thread0()` that loads a value of zero from `x` must complete before `thread1()` stores to `y`, so that instance must also load a value of zero from `y`. Similarly, any instance of `thread0()` that loads a value of one from `y` must have started after the `synchronize_rcu()` started, and must therefore also load a value of one from `x`. Therefore, the outcome: 

```
(r1 == 0 && r2 == 1)
```

cannot happen. 

This scenario resembles one of the first uses of RCU in [DYNIX/ptx](<http://en.wikipedia.org/wiki/DYNIX>), which managed a distributed lock manager's transition into a state suitable for handling recovery from node failure, more or less as follows: 

> 

```
>      1 #define STATE_NORMAL        0
>      2 #define STATE_WANT_RECOVERY 1
>      3 #define STATE_RECOVERING    2
>      4 #define STATE_WANT_NORMAL   3
>      5 
>      6 int state = STATE_NORMAL;
>      7 
>      8 void do_something_dlm(void)
>      9 {
>     10   int state_snap;
>     11 
>     12   rcu_read_lock();
>     13   state_snap = READ_ONCE(state);
>     14   if (state_snap == STATE_NORMAL)
>     15     do_something();
>     16   else
>     17     do_something_carefully();
>     18   rcu_read_unlock();
>     19 }
>     20 
>     21 void start_recovery(void)
>     22 {
>     23   WRITE_ONCE(state, STATE_WANT_RECOVERY);
>     24   synchronize_rcu();
>     25   WRITE_ONCE(state, STATE_RECOVERING);
>     26   recovery();
>     27   WRITE_ONCE(state, STATE_WANT_NORMAL);
>     28   synchronize_rcu();
>     29   WRITE_ONCE(state, STATE_NORMAL);
>     30 }
>
```

**Quick Quiz 2** : Why is the `synchronize_rcu()` on line 28 needed?   
Answer

The RCU read-side critical section in `do_something_dlm()` works with the `synchronize_rcu()` in `start_recovery()` to guarantee that `do_something()` never runs concurrently with `recovery()`, but with little or no synchronization overhead in `do_something_dlm()`. 

In order to avoid fatal problems such as deadlocks, an RCU read-side critical section must not contain calls to `synchronize_rcu()`. Similarly, an RCU read-side critical section must not contain anything that waits, directly or indirectly, on completion of an invocation of `synchronize_rcu()`. 

Although RCU's grace-period guarantee is useful in and of itself, with [quite a few use cases](<https://lwn.net/Articles/573497/>), it would be good to be able to use RCU to coordinate read-side access to linked data structures. For this, the grace-period guarantee is not sufficient, as can be seen in function `add_gp_buggy()` below. We will look at the reader's code later, but in the meantime, just think of the reader as locklessly picking up the `gp` pointer, and, if the value loaded is non-`NULL`, locklessly accessing the `->a` and `->b` fields. 

> 

```
>      1 bool add_gp_buggy(int a, int b)
>      2 {
>      3   p = kmalloc(sizeof(*p), GFP_KERNEL);
>      4   if (!p)
>      5     return -ENOMEM;
>      6   spin_lock(&gp_lock);
>      7   if (rcu_access_pointer(gp)) {
>      8     spin_unlock(&gp_lock);
>      9     return false;
>     10   }
>     11   p->a = a;
>     12   p->b = a;
>     13   gp = p; /* ORDERING BUG */
>     14   spin_unlock(&gp_lock);
>     15   return true;
>     16 }
>
```

The problem is that both the compiler and weakly ordered CPUs are within their rights to reorder this code as follows: 

> 

```
>      1 bool add_gp_buggy_optimized(int a, int b)
>      2 {
>      3   p = kmalloc(sizeof(*p), GFP_KERNEL);
>      4   if (!p)
>      5     return -ENOMEM;
>      6   spin_lock(&gp_lock);
>      7   if (rcu_access_pointer(gp)) {
>      8     spin_unlock(&gp_lock);
>      9     return false;
>     10   }
>     **11   gp = p; /* ORDERING BUG */
>     12   p- >a = a;
>     13   p->b = a;**
>     14   spin_unlock(&gp_lock);
>     15   return true;
>     16 }
>
```

If an RCU reader fetches `gp` just after `add_gp_buggy_optimized` executes line 11, it will see garbage in the `->a` and `->b` fields. And this is but one of many ways in which compiler and hardware optimizations could cause trouble. Therefore, we clearly need some way to prevent the compiler and the CPU from reordering in this manner, which brings us to the publish-subscribe guarantee discussed in the next section. 

#### Publish-subscribe guarantee

RCU's publish-subscribe guarantee allows data to be inserted into a linked data structure without disrupting RCU readers. The updater uses `rcu_assign_pointer()` to insert the new data, and readers use `rcu_dereference()` to access data, whether new or old. The following shows an example of insertion: 

> 

```
>      1 bool add_gp(int a, int b)
>      2 {
>      3   p = kmalloc(sizeof(*p), GFP_KERNEL);
>      4   if (!p)
>      5     return -ENOMEM;
>      6   spin_lock(&gp_lock);
>      7   if (rcu_access_pointer(gp)) {
>      8     spin_unlock(&gp_lock);
>      9     return false;
>     10   }
>     11   p->a = a;
>     12   p->b = a;
>     13   rcu_assign_pointer(gp, p);
>     14   spin_unlock(&gp_lock);
>     15   return true;
>     16 }
>
```

**Quick Quiz 3** : But `rcu_assign_pointer()` does nothing to prevent the two assignments to `p->a` and `p->b` from being reordered. Can't that also cause problems?   
Answer

The `rcu_assign_pointer()` on line 13 is conceptually equivalent to a simple assignment statement, but also guarantees that its assignment will happen after any prior assignments, including the two in lines 11 and 12, similar to the C11 `memory_order_release` store operation. It also prevents any number of "interesting" compiler optimizations, for example, the use of `gp` as a scratch location immediately preceding the assignment. 

It is tempting to assume that the reader need not do anything special to control its accesses to the RCU-protected data, as shown in `do_something_gp_buggy()` below: 

> 

```
>      1 bool do_something_gp_buggy(void)
>      2 {
>      3   rcu_read_lock();
>      4   p = gp;  /* OPTIMIZATIONS GALORE!!! */
>      5   if (p) {
>      6     do_something(p->a, p->b);
>      7     rcu_read_unlock();
>      8     return true;
>      9   }
>     10   rcu_read_unlock();
>     11   return false;
>     12 }
>
```

However, this temptation must be resisted because there are a surprisingly large number of ways that the compiler (to say nothing of [DEC Alpha CPUs](<http://h71000.www7.hp.com/wizard/wiz_2637.html>)) can trip this code up. For but one example, if the compiler were short of registers, it might choose to refetch from `gp` rather than keeping a separate copy in `p` as follows: 

> 

```
>      1 bool do_something_gp_buggy_optimized(void)
>      2 {
>      3   rcu_read_lock();
>      4   if (gp) { /* OPTIMIZATIONS GALORE!!! */
>      5     do_something(gp->a, gp->b);
>      6     rcu_read_unlock();
>      7     return true;
>      8   }
>      9   rcu_read_unlock();
>     10   return false;
>     11 }
>
```

If this function ran concurrently with a series of updates that replaced the current structure with a new one, the fetches of `gp->a` and `gp->b` might well come from two different structures, which could cause serious confusion. To prevent this (and much else besides), `do_something_gp()` uses `rcu_dereference()` to fetch from `gp`: 

> 

```
>      1 bool do_something_gp(void)
>      2 {
>      3   rcu_read_lock();
>      4   p = rcu_dereference(gp);
>      5   if (p) {
>      6     do_something(p->a, p->b);
>      7     rcu_read_unlock();
>      8     return true;
>      9   }
>     10   rcu_read_unlock();
>     11   return false;
>     12 }
>
```

The `rcu_dereference()` uses volatile casts and (for DEC Alpha) memory barriers in the Linux kernel. Should a [high-quality implementation of C11 `memory_order_consume` [PDF]](<http://www.rdrop.com/users/paulmck/RCU/consume.2015.07.13a.pdf>) ever appear, then `rcu_dereference()` could be implemented as a `memory_order_consume` load. Regardless of the exact implementation, a pointer fetched by `rcu_dereference()` may not be used outside of the outermost RCU read-side critical section containing that `rcu_dereference()`, unless protection of the corresponding data element has been passed from RCU to some other synchronization mechanism, most commonly locking or [reference counting](<https://www.kernel.org/doc/Documentation/RCU/rcuref.txt>). 

In short, updaters use `rcu_assign_pointer()` and readers use `rcu_dereference()`, and these two RCU API elements work together to ensure that readers have a consistent view of newly added data elements. 

Of course, it is also necessary to remove elements from RCU-protected data structures, for example, using the following process: 

  1. Remove the data element from the enclosing structure. 
  2. Wait for all pre-existing RCU read-side critical sections to complete (because only pre-existing readers can possibly have a reference to the newly removed data element). 
  3. At this point, only the updater has a reference to the newly removed data element, so it can safely reclaim the data element, for example, by passing it to `kfree()`. 
This process is implemented by `remove_gp_synchronous()`: 

> 

```
>      1 bool remove_gp_synchronous(void)
>      2 {
>      3   struct foo *p;
>      4 
>      5   spin_lock(&gp_lock);
>      6   p = rcu_access_pointer(gp);
>      7   if (!p) {
>      8     spin_unlock(&gp_lock);
>      9     return false;
>     10   }
>     11   rcu_assign_pointer(gp, NULL);
>     12   spin_unlock(&gp_lock);
>     13   synchronize_rcu();
>     14   kfree(p);
>     15   return true;
>     16 }
>
```

This function is straightforward, with line 13 waiting for a grace period before line 14 frees the old data element. This waiting ensures that readers will reach line 7 of `do_something_gp()` before the data element referenced by `p` is freed. The `rcu_access_pointer()` on line 6 is similar to `rcu_dereference()`, except that: 

**Quick Quiz 4** : Without the `rcu_dereference()` or the `rcu_access_pointer()`, what destructive optimizations might the compiler make use of?   
Answer

  1. The value returned by `rcu_access_pointer()` cannot be dereferenced. If you want to access the value pointed to as well as the pointer itself, use `rcu_dereference()` instead of `rcu_access_pointer()`. 
  2. The call to `rcu_access_pointer()` need not be protected. In contrast, `rcu_dereference()` must either be within an RCU read-side critical section or in a code segment where the pointer cannot change, for example, in code protected by the corresponding update-side lock. 

This simple linked-data-structure scenario clearly demonstrates the need for RCU's stringent memory-ordering guarantees on systems with more than one CPU: 

  1. **Quick Quiz 5** : Given that multiple CPUs can start RCU read-side critical sections at any time without any ordering whatsoever, how can RCU possibly tell whether or not a given RCU read-side critical section starts before a given instance of `synchronize_rcu()`?   
Answer

Each CPU that has an RCU read-side critical section that begins before `synchronize_rcu()` starts is guaranteed to execute a full memory barrier between the time that the RCU read-side critical section ends and the time that `synchronize_rcu()` returns. Without this guarantee, a pre-existing RCU read-side critical section might hold a reference to the newly removed `struct foo` after the `kfree()` on line 14 of `remove_gp_synchronous()`. 
  2. Each CPU that has an RCU read-side critical section that ends after `synchronize_rcu()` returns is guaranteed to execute a full memory barrier between the time that `synchronize_rcu()` begins and the time that the RCU read-side critical section begins. Without this guarantee, a later RCU read-side critical section running after the `kfree()` on line 14 of `remove_gp_synchronous()` might later run `do_something_gp()` and find the newly deleted `struct foo`. 
  3. **Quick Quiz 6** : The first and second guarantees require unbelievably strict ordering! Are all these memory barriers _really_ required?   
Answer

If the task invoking `synchronize_rcu()` remains on a given CPU, then that CPU is guaranteed to execute a full memory barrier sometime during the execution of `synchronize_rcu()`. This guarantee ensures that the `kfree()` on line 14 of `remove_gp_synchronous()` really does execute after the removal on line 11\. 
  4. If the task invoking `synchronize_rcu()` migrates among a group of CPUs during that invocation, then each of the CPUs in that group is guaranteed to execute a full memory barrier sometime during the execution of `synchronize_rcu()`. This guarantee also ensures that the `kfree()` on line 14 of `remove_gp_synchronous()` really does execute after the removal on line 11, but also in the case where the thread executing the `synchronize_rcu()` migrates in the meantime. 

In short, RCU's publish-subscribe guarantee is provided by the combination of `rcu_assign_pointer()` and `rcu_dereference()`. This guarantee allows data elements to be safely added to RCU-protected linked data structures without disrupting RCU readers. This guarantee can be used in combination with the grace-period guarantee to also allow data elements to be removed from RCU-protected linked data structures, again without disrupting RCU readers. 

This guarantee was only partially premeditated. DYNIX/ptx used an explicit memory barrier for publication, but had nothing resembling `rcu_dereference()` for subscription, nor did it have anything resembling the `smp_read_barrier_depends()` that was later subsumed into `rcu_dereference()`. The need for these operations made itself known quite suddenly at a late-1990s meeting with the DEC Alpha architects, back in the days when DEC was still a freestanding company. It took the Alpha architects a good hour to convince me that any sort of barrier would ever be needed, and it then took me a good _two_ hours to convince them that their documentation did not make this point clear. More recent work with the C and C++ standards committees have provided much education on tricks and traps from the compiler. In short, compilers were much less tricky in the early 1990s, but in 2015, don't even think about omitting `rcu_dereference()`! 

#### RCU primitives guaranteed to execute unconditionally

The common-case RCU primitives are unconditional. They are invoked, they do their job, and they return, with no possibility of error and no need to retry. This is a key RCU design philosophy. 

However, this philosophy is pragmatic rather than pigheaded. If someone comes up with a good justification for a particular conditional RCU primitive, it might well be implemented and added. After all, this guarantee was reverse-engineered, not premeditated. The unconditional nature of the RCU primitives was initially an accident of implementation, and later experience with synchronization primitives with conditional primitives caused me to elevate this accident to a guarantee. Therefore, the justification for adding a conditional primitive to RCU would need to be based on detailed and compelling use cases. 

#### Guaranteed read-to-write upgrade

**Quick Quiz 7** : But how does the upgrade-to-write operation exclude other readers?   
Answer

As far as RCU is concerned, it is always possible to carry out an update within an RCU read-side critical section. For example, that RCU read-side critical section might search for a given data element, and then might acquire the update-side spinlock in order to update that element, all while remaining in that RCU read-side critical section. Of course, it is necessary to exit the RCU read-side critical section before invoking `synchronize_rcu()`, however, this inconvenience can be avoided through use of the `call_rcu()` and `kfree_rcu()` API members described later in this document. 

This guarantee allows lookup code to be shared between read-side and update-side code, and was premeditated, appearing in the earliest DYNIX/ptx RCU documentation. 

#### Fundamental non-requirements

RCU provides extremely lightweight readers, and its read-side guarantees, though quite useful, are correspondingly lightweight. It is therefore all too easy to assume that RCU is guaranteeing more than it really is. Of course, the list of things that RCU does not guarantee is infinitely long, however, the following sections list a few non-guarantees that have caused confusion. Except where otherwise noted, these non-guarantees were premeditated. 

  1. Readers Impose Minimal Ordering
  2. Readers Do Not Exclude Updaters
  3. Updaters Only Wait For Old Readers
  4. Grace Periods Don't Partition Read-Side Critical Sections
  5. Read-Side Critical Sections Don't Partition Grace Periods
  6. Disabling Preemption Does Not Block Grace Periods 

#### Readers impose minimal ordering

Reader-side markers such as `rcu_read_lock()` and `rcu_read_unlock()` provide absolutely no ordering guarantees except through their interaction with the grace-period APIs such as `synchronize_rcu()`. To see this, consider the following pair of threads: 

> 

```
>      1 void thread0(void)
>      2 {
>      3   rcu_read_lock();
>      4   WRITE_ONCE(x, 1);
>      5   rcu_read_unlock();
>      6   rcu_read_lock();
>      7   WRITE_ONCE(y, 1);
>      8   rcu_read_unlock();
>      9 }
>     10 
>     11 void thread1(void)
>     12 {
>     13   rcu_read_lock();
>     14   r1 = READ_ONCE(y);
>     15   rcu_read_unlock();
>     16   rcu_read_lock();
>     17   r2 = READ_ONCE(x);
>     18   rcu_read_unlock();
>     19 }
>
```

After `thread0()` and `thread1()` execute concurrently, it is quite possible to have: 

```
(r1 == 1 && r2 == 0)
```

**Quick Quiz 8** : Can't the compiler also reorder this code?   
Answer

(i.e. `y` appears to have been assigned before `x`) which would not be possible if `rcu_read_lock()` and `rcu_read_unlock()` had much in the way of ordering properties. But they do not, so the CPU is within its rights to do significant reordering. This is by design: Any significant ordering constraints would slow down these fast-path APIs. 

#### Readers do not exclude updaters

Neither `rcu_read_lock()` nor `rcu_read_unlock()` exclude updates. All they do is to prevent grace periods from ending. The following example illustrates this: 

> 

```
>      1 void thread0(void)
>      2 {
>      3   rcu_read_lock();
>      4   r1 = READ_ONCE(y);
>      5   if (r1) {
>      6     do_something_with_nonzero_x();
>      7     r2 = READ_ONCE(x);
>      8     WARN_ON(!r2); /* BUG!!! */
>      9   }
>     10   rcu_read_unlock();
>     11 }
>     12 
>     13 void thread1(void)
>     14 {
>     15   spin_lock(&my_lock);
>     16   WRITE_ONCE(x, 1);
>     17   WRITE_ONCE(y, 1);
>     18   spin_unlock(&my_lock);
>     19 }
>
```

If the `thread0()` function's `rcu_read_lock()` excluded the `thread1()` function's update, the `WARN_ON()` could never fire. But the fact is that `rcu_read_lock()` does not exclude much of anything aside from subsequent grace periods, of which `thread1()` has none, so the `WARN_ON()` can and does fire. 

#### Updaters only wait for old readers

**Quick Quiz 9** : Suppose that synchronize_rcu() did wait until all readers had completed. Would the updater be able to rely on this?   
Answer

It might be tempting to assume that after `synchronize_rcu()` completes, there are no readers executing. This temptation must be avoided because new readers can start immediately after `synchronize_rcu()` starts, and `synchronize_rcu()` is under no obligation to wait for these new readers. 

####  Grace periods don't partition read-side critical sections

One might assume that if any part of one RCU read-side critical section precedes a given grace period, and if any part of another RCU read-side critical section follows that same grace period, then all of the first RCU read-side critical section must precede all of the second. However, this just isn't the case: a single grace period does not partition the set of RCU read-side critical sections. An example of this situation can be illustrated as follows, where `a`, `b`, and `c` are initially all zero: 

> 

```
>      1 void thread0(void)
>      2 {
>      3   rcu_read_lock();
>      4   WRITE_ONCE(a, 1);
>      5   WRITE_ONCE(b, 1);
>      6   rcu_read_unlock();
>      7 }
>      8 
>      9 void thread1(void)
>     10 {
>     11   r1 = READ_ONCE(a);
>     12   synchronize_rcu();
>     13   WRITE_ONCE(c, 1);
>     14 }
>     15 
>     16 void thread2(void)
>     17 {
>     18   rcu_read_lock();
>     19   r2 = READ_ONCE(b);
>     20   r3 = READ_ONCE(c);
>     21   rcu_read_unlock();
>     22 }
>
```

It turns out that the outcome: 

```
(r1 == 1 && r2 == 0 && r3 == 1)
```

is entirely possible. The following figure show how this can happen, with each circled `QS` indicating the point at which RCU recorded a _quiescent state_ for each thread, that is, a state in which RCU knows that the thread cannot be in the midst of an RCU read-side critical section that started before the current grace period: 

> ![GPpartitionReaders1.png](https://static.lwn.net/images/2015/rcu/GPpartitionReaders1.png)

If it is necessary to partition RCU read-side critical sections in this manner, it is necessary to use two grace periods, where the first grace period is known to end before the second grace period starts: 

> 

```
>      1 void thread0(void)
>      2 {
>      3   rcu_read_lock();
>      4   WRITE_ONCE(a, 1);
>      5   WRITE_ONCE(b, 1);
>      6   rcu_read_unlock();
>      7 }
>      8 
>      9 void thread1(void)
>     10 {
>     11   r1 = READ_ONCE(a);
>     12   synchronize_rcu();
>     13   WRITE_ONCE(c, 1);
>     14 }
>     15 
>     16 void thread2(void)
>     17 {
>     18   r2 = READ_ONCE(c);
>     19   synchronize_rcu();
>     20   WRITE_ONCE(d, 1);
>     21 }
>     22 
>     23 void thread3(void)
>     24 {
>     25   rcu_read_lock();
>     26   r3 = READ_ONCE(b);
>     27   r4 = READ_ONCE(d);
>     28   rcu_read_unlock();
>     29 }
>
```

Here, if `(r1 == 1)`, then `thread0()`'s write to `b` must happen before the end of `thread1()`'s grace period. If in addition `(r4 == 1)`, then `thread3()`'s read from `b` must happen after the beginning of `thread2()`'s grace period. If it is also the case that `(r2 == 1)`, then the end of `thread1()`'s grace period must precede the beginning of `thread2()`'s grace period. This means that the two RCU read-side critical sections cannot overlap, guaranteeing that `(r3 == 1)`. As a result, the outcome: 

```
(r1 == 1 && r2 == 1 && r3 == 0 && r4 == 1)
```

cannot happen. 

This non-requirement was also non-premeditated, but became apparent when studying RCU's interaction with memory ordering. 

####  Read-side critical sections don't partition grace periods

It is also tempting to assume that if an RCU read-side critical section happens between a pair of grace periods, then those grace periods cannot overlap. However, this temptation leads nowhere good, as can be illustrated by the following, with all variables initially zero: 

> 

```
>      1 void thread0(void)
>      2 {
>      3   rcu_read_lock();
>      4   WRITE_ONCE(a, 1);
>      5   WRITE_ONCE(b, 1);
>      6   rcu_read_unlock();
>      7 }
>      8 
>      9 void thread1(void)
>     10 {
>     11   r1 = READ_ONCE(a);
>     12   synchronize_rcu();
>     13   WRITE_ONCE(c, 1);
>     14 }
>     15 
>     16 void thread2(void)
>     17 {
>     18   rcu_read_lock();
>     19   WRITE_ONCE(d, 1);
>     20   r2 = READ_ONCE(c);
>     21   rcu_read_unlock();
>     22 }
>     23 
>     24 void thread3(void)
>     25 {
>     26   r3 = READ_ONCE(d);
>     27   synchronize_rcu();
>     28   WRITE_ONCE(e, 1);
>     29 }
>     30 
>     31 void thread4(void)
>     32 {
>     33   rcu_read_lock();
>     34   r4 = READ_ONCE(b);
>     35   r5 = READ_ONCE(e);
>     36   rcu_read_unlock();
>     37 }
>
```

In this case, the outcome: 

```
(r1 == 1 && r2 == 1 && r3 == 1 && r4 == 0 && r5 == 1)
```

is entirely possible, as illustrated below: 

> ![ReadersPartitionGP1.svg](https://static.lwn.net/images/2015/rcu/ReadersPartitionGP1.png)

**Quick Quiz 10** : How long a sequence of grace periods, each separated by an RCU read-side critical section, would be required to partition the RCU read-side critical sections at the beginning and end of the chain?   
Answer

Again, an RCU read-side critical section can overlap almost all of a given grace period, just so long as it does not overlap the entire grace period. As a result, an RCU read-side critical section cannot partition a pair of RCU grace periods. 

####  Disabling preemption does not block grace periods

There was a time when disabling preemption on any given CPU would block subsequent grace periods. However, this was an accident of implementation and is not a requirement. And in the current Linux-kernel implementation, disabling preemption on a given CPU in fact does not block grace periods, as Oleg Nesterov [demonstrated](<http://marc.info/?l=linux-kernel&m=143431078202914&w=2>). 

If you need a preempt-disable region to block grace periods, you need to add `rcu_read_lock()` and `rcu_read_unlock()`, for example as follows: 

> 

```
>      1 preempt_disable();
>      2 rcu_read_lock();
>      3 do_something();
>      4 rcu_read_unlock();
>      5 preempt_enable();
>      6 
>      7 /* Spinlocks implicitly disable preemption. */
>      8 spin_lock(&mylock);
>      9 rcu_read_lock();
>     10 do_something();
>     11 rcu_read_unlock();
>     12 spin_unlock(&mylock);
>
```

In theory, you could enter the RCU read-side critical section first, but it is more efficient to keep the entire RCU read-side critical section contained in the preempt-disable region as shown above. Of course, RCU read-side critical sections that extend outside of preempt-disable regions will work correctly, but such critical sections can be preempted, which forces `rcu_read_unlock()` to do more work. And no, this is _not_ an invitation to enclose all of your RCU read-side critical sections within preempt-disable regions, because doing so would degrade realtime response. 

This non-requirement appeared with preemptible RCU. If you need a grace period that waits on non-preemptible code regions, use RCU-sched. 

Here ends part 1; the remaining two parts will be published in the coming weeks. 

####  Answers to quick quizzes

**Quick Quiz 1** : Wait a minute! You said that updaters can make useful forward progress concurrently with readers, but pre-existing readers will block `synchronize_rcu()`!!! Just who are you trying to fool??? 

**Answer** : First, if updaters do not wish to be blocked by readers, they can use `call_rcu()` or `kfree_rcu()`, which will be discussed later. Second, even when using `synchronize_rcu()`, the other update-side code does run concurrently with readers, whether pre-existing or not. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : Why is the `synchronize_rcu()` on line 28 needed? 

**Answer** : Without that extra grace period, memory reordering could result in `do_something_dlm()` executing `do_something()` concurrently with the last bits of `recovery()`. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : But `rcu_assign_pointer()` does nothing to prevent the two assignments to `p->a` and `p->b` from being reordered. Can't that also cause problems? 

**Answer** : No, it cannot. The readers cannot see either of these two fields until the assignment to `gp`, by which time both fields are fully initialized. So reordering the assignments to `p->a` and `p->b` cannot possibly cause any problems. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : Without the `rcu_dereference()` or the `rcu_access_pointer()`, what destructive optimizations might the compiler make use of? 

**Answer** : Let's start with what happens to `do_something_gp()` if it fails to use `rcu_dereference()`. It could reuse a value formerly fetched from this same pointer. It could also fetch the pointer from `gp` in a byte-at-a-time manner, resulting in _load tearing_ , in turn resulting in a bytewise mash-up of two distinct pointer values. It might even use value-speculation optimizations, where it makes a wrong guess, but by the time it gets around to checking the value, an update has changed the pointer to match the wrong guess. Too bad about any dereferences that returned pre-initialization garbage in the meantime! 

For `remove_gp_synchronous()`, as long as all modifications to `gp` are carried out while holding `gp_lock`, the above optimizations are harmless. However, with `CONFIG_SPARSE_RCU_POINTER=y`, `sparse` will complain if you define `gp` with `__rcu` and then access it without using either `rcu_access_pointer()` or `rcu_dereference()`. 

**Back to Quick Quiz 4**.

**Quick Quiz 5** : Given that multiple CPUs can start RCU read-side critical sections at any time without any ordering whatsoever, how can RCU possibly tell whether or not a given RCU read-side critical section starts before a given instance of `synchronize_rcu()`? 

**Answer** : If RCU cannot tell whether or not a given RCU read-side critical section starts before a given instance of `synchronize_rcu()`, then it must assume that the RCU read-side critical section started first. In other words, a given instance of `synchronize_rcu()` can avoid waiting on a given RCU read-side critical section only if it can prove that `synchronize_rcu()` started first. 

**Back to Quick Quiz 5**.

**Quick Quiz 6** : The first and second guarantees require unbelievably strict ordering! Are all these memory barriers _really_ required? 

**Answer** : Yes, they really are required. To see why the first guarantee is required, consider the following sequence of events: 

  1. CPU 1: `rcu_read_lock()`
  2. CPU 1: `q = rcu_dereference(gp); /* Very likely to return p. */`
  3. CPU 0: `list_del_rcu(p);`
  4. CPU 0: `synchronize_rcu()` starts. 
  5. CPU 1: `do_something_with(q->a); /* No smp_mb(), so might happen after kfree(). */`
  6. CPU 1: `rcu_read_unlock()`
  7. CPU 0: `synchronize_rcu()` returns. 
  8. CPU 0: `kfree(p);` 

Therefore, there absolutely must be a full memory barrier between the end of the RCU read-side critical section and the end of the grace period. 

The sequence of events demonstrating the necessity of the second rule is roughly similar: 

  1. CPU 0: `list_del_rcu(p);`
  2. CPU 0: `synchronize_rcu()` starts. 
  3. CPU 1: `rcu_read_lock()`
  4. CPU 1: `q = rcu_dereference(gp); /* Might return p if no memory barrier. */`
  5. CPU 0: `synchronize_rcu()` returns. 
  6. CPU 0: `kfree(p);`
  7. CPU 1: `do_something_with(q->a); /* Boom!!! */`
  8. CPU 1: `rcu_read_unlock()` 

And similarly, without a memory barrier between the beginning of the grace period and the beginning of the RCU read-side critical section, CPU 1 might end up accessing the freelist. 

The "as if" rule of course applies, so that any implementation that acts as if the appropriate memory barriers were in place is a correct implementation. That said, it is much easier to fool yourself into believing that you have adhered to the as-if rule than it is to actually adhere to it! 

**Back to Quick Quiz 6**.

**Quick Quiz 7** : But how does the upgrade-to-write operation exclude other readers? 

**Answer** : It doesn't, just like normal RCU updates, which also do not exclude RCU readers. 

**Back to Quick Quiz 7**.

**Quick Quiz 8** : Can't the compiler also reorder this code? 

**Answer** : No, the volatile casts in `READ_ONCE()` and `WRITE_ONCE()` prevent reordering in this particular case. 

**Back to Quick Quiz 8**.

**Quick Quiz 9** : Suppose that synchronize_rcu() did wait until all readers had completed. Would the updater be able to rely on this? 

**Answer** : No. Even if `synchronize_rcu()` were to wait until all readers had completed, a new reader might start immediately after `synchronize_rcu()` completed. Therefore, the code following `synchronize_rcu()` cannot rely on there being no readers in any case. 

**Back to Quick Quiz 9**.

**Quick Quiz 10** : How long a sequence of grace periods, each separated by an RCU read-side critical section, would be required to partition the RCU read-side critical sections at the beginning and end of the chain? 

**Answer** : In theory, an infinite number. In practice, an unknown number that is sensitive to both implementation details and timing considerations. Therefore, even in practice, RCU users must abide by the theoretical rather than the practical answer. 

**Back to Quick Quiz 10**.
