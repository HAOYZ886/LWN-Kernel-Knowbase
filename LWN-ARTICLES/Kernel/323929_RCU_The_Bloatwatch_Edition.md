---
title: "RCU: The Bloatwatch Edition"
url: https://lwn.net/Articles/323929/
date: "March 17, 2009"
category: "Read-copy-update"
author: "March 17, 2009 This article was contributed by Paul McKenney"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

March 17, 2009

This article was contributed by Paul McKenney

### Introduction

Read-copy update (RCU) is a synchronization mechanism that was added to the Linux kernel in October of 2002. RCU improves scalability by allowing readers to execute concurrently with writers. In contrast, conventional locking primitives require that readers wait for ongoing writers and vice versa. RCU ensures read coherence by maintaining multiple versions of data structures and ensuring that they are not freed up until all pre-existing read-side critical sections complete. RCU relies on efficient and scalable mechanisms for publishing and reading new versions of an object, and also for deferring the collection of old versions. These mechanisms distribute the work among read and update paths in such a way as to make read paths extremely fast. In some cases (non-preemptable kernels), RCU's read-side primitives have zero overhead. RCU updates can be expensive, so RCU is in general best-suited to read-mostly workloads. 

Although Classic RCU's memory footprint has been acceptable, [hierarchical RCU](<http://lwn.net/Articles/305782/>) has a memory footprint that is considerably larger. I was surprised to find that this additional memory consumption did not greatly concern the embedded Linux people I talked to, but then again, I certainly do not know everyone in the embedded Linux community, and the complexity of both [hierarchical RCU](<http://lwn.net/Articles/305782/>), [preemptable RCU](<http://lwn.net/Articles/253651/>), and [the dynticks interface](<http://lwn.net/Articles/279077/>) made the thought of a near-minimal kernel-capable RCU with a classic RCU API more interesting. In addition, a recent [ show of hands](<http://paulmck.livejournal.com/2280.html>) at [linux.conf.au](<http://linux.conf.au>) indicated that there is still interest in running Linux on small-memory single-CPU systems. Discussions with Josh Triplett identified an attractive optimization for the uniprocessor case, and so "RCU: The Bloatwatch Edition" was born. 

Contents:

  1. Review of RCU Fundamentals
  2. Design of Tiny RCU
  3. Tiny-RCU Code Walkthrough
  4. Testing
  5. Memory Footprint

These sections are followed by concluding remarks and the answers to the Quick Quizzes. 

###  Review of RCU Fundamentals

This section is quite similar to its counterpart in the description of [hierarchical RCU](<http://lwn.net/Articles/305782>). People familiar with RCU semantics may wish to proceed directly to the next section. 

In its most basic form, RCU is a way of waiting for things to finish. Of course, there are a great many other ways of waiting for things to finish, including reference counts, reader-writer locks, hashed locks, events, and so on. The great advantage of RCU is that it can wait for each of (say) 20,000 different things without having to explicitly track each and every one of them, and without having to worry about the performance degradation, scalability limitations, complex deadlock scenarios, and memory-leak hazards that are inherent in schemes using explicit tracking. 

In RCU's case, the things waited on are called "RCU read-side critical sections". An RCU read-side critical section starts with an `rcu_read_lock()` primitive, and ends with a corresponding `rcu_read_unlock()` primitive. RCU read-side critical sections can be nested, and may contain pretty much any code, as long as that code does not explicitly block or sleep. If you abide by these conventions, you can use RCU to wait for pretty much any desired piece of code to complete. 

**Quick Quiz 1** : But what if I need my RCU read-side critical section to sleep and block? 

RCU accomplishes this feat by indirectly determining when these other things have finished, as has been described elsewhere for [ Classic RCU](<http://www.rdrop.com/users/paulmck/RCU/whatisRCU.html>) and [realtime RCU](<http://lwn.net/Articles/253651/>). 

In particular, as shown in the following figure, RCU is a way of waiting for pre-existing RCU read-side critical sections to completely finish, including memory operations executed by those critical sections. 

> ![Grace periods extend to contain pre-existing RCU read-side critical
sections.](/images/ns/kernel/brcu/GracePeriodGood.png)

However, note that RCU read-side critical sections that begin after the beginning of a given grace period can and will extend beyond the end of that grace period. 

The following section gives a very high-level view of how the Tiny RCU implementation operates. 

###  Design of Tiny RCU

The key restriction that enables a smaller and simpler RCU implementation is `CONFIG_SMP=n`, which means that any time the sole CPU passes through a quiescent state, a grace period has elapsed. In principle, the scheduler could simply invoke all pending RCU callbacks on each context switch, but in practice it is wise to maintain a degree of isolation between the scheduler and RCU. An arms-length relationship allows RCU callbacks to invoke appropriate scheduler functions (e.g., `wake_up()`) without potential deadlock issues. Therefore, the RCU core runs in softirq context. 

This uniprocessor approach also simplifies the data structure, so that each flavor of RCU (`rcu_ctrlblk` and `rcu_bh_ctrlblk`) has the following structure: 

```
1 struct rcu_ctrlblk {
      2   long completed;
      3   struct rcu_head *rcucblist;
      4   struct rcu_head **donetail;
      5   struct rcu_head **curtail;
      6 };
```

The `->completed` field is required only for the `rcu_batches_completed()` and `rcu_batches_completed_bh()` interfaces used by the RCU torture tests. The `->rcucblist` field is the header for the list of RCU callbacks (`rcu_head` structures), the `->donetail` field references the `next` pointer of the last `rcu_head` structure in the list whose grace period has completed, and the `->curtail` field references the `next` pointer of the last `rcu_head` structure in the list. 

The following figure shows a sample callback list that has two callbacks ready to invoke and a third callback still waiting for a grace period (or, equivalently on a uniprocessor system, for a quiescent state): 

> ![Tiny RCU callback list](/images/ns/kernel/brcu/TinyRCUCallbacks.png)

**Quick Quiz 2** : But we should be able to simply execute _all_ callbacks at each quiescent state! So why bother with separate `->donetail` and `->curtail` sublists? 

![Schematic of Tree RCU](/images/ns/kernel/brcu/RCUbweBlock-sm.png) A block diagram of tiny RCU appears to the right; see [this page](<https://lwn.net/Articles/323939/>) for a full-size version. 

The blue rectangles in this diagram represent functions making up tiny RCU, the blue rectangles with rounded corners represent data structures within tiny RCU, and the red rectangles represent other parts of the Linux kernel. Solid arrows represent function-call interfaces, while dashed arrows represent indirect invocation. The RCU read-side and list-manipulation primitives are not shown, nor are the interfaces specific to the "rcutorture" test module. 

Like classic and hierarchical RCU, tiny RCU's grace-period detection is driven by context switches, the scheduling-clock interrupt (`scheduler_tick()`), and the idle loop. The scheduling-clock interrupt handler and idle loops contain code as follows: 

```
if (rcu_pending(cpu))
    	rcu_check_callbacks(cpu, 0);
```

So if `rcu_pending()` indicates that the RCU core has any work to do, `rcu_check_callbacks()` is invoked, which in turn checks to see if the CPU is currently in a quiescent state, invoking either or both of `rcu_qsctr_inc()` and `rcu_bh_qsctr_inc()` as appropriate. These in turn invoke `rcu_qsctr_help()`, which, if there are RCU callbacks present, updates the callback lists to indicate that their grace period has elapsed and returns 1 to tell the caller to invoke `raise_softirq()`. At some later time, `rcu_process_callbacks()` will be invoked from softirq context, which, via `__rcu_process_callbacks()`, invokes all callbacks in `rcu_ctrlblk` and `rcu_bh_ctrlblk`. 

Tiny RCU also interfaces to dynticks, albeit in a slightly different way than do the classic, preemptable, and hierarchical RCU implementations. Because there is but a single CPU, entering dynticks-idle mode is both a quiescent state and a grace period, allowing `rcu_qsctr_inc()` to directly invoke `raise_softirq()`. This direct invocation in turn means that `rcu_needs_cpu()` can simply return zero, since any callbacks are processed on the way into dynticks-idle state. 

The `call_rcu()` and `call_rcu_bh()` interfaces queue callbacks via the `__call_rcu()` helper function. The `synchronize_rcu()` interface is a special case that will be described in the code walkthrough in the next section. 

Interestingly enough, the general shape of the above block diagram applies to all RCU implementations. Of course, the processing is more complex in the `CONFIG_SMP` case, particularly in the `__rcu_process_callbacks()` area. 

###  Tiny-RCU Code Walkthrough

This code walkthrough starts with tiny RCU's update-side API, then discusses the grace-period machinery, and finally the dynticks interface. 

#### Update-Side API

The code for `synchronize_rcu()` is as follows: 

```
1 void synchronize_rcu(void)
      2 {
      3   unsigned long flags;
      4 
      5   local_irq_save(flags);
      6   rcu_ctrlblk.completed++;
      7   local_irq_restore(flags);
      8 }
```

This code merely increments the `->completed` field with interrupts disabled. This works because `synchronize_rcu()` may only be invoked when it is legal to block, in other words, `synchronize_rcu()` cannot be called from within an RCU read-side critical section. Therefore, anywhere `synchronize_rcu()` may legally be invoked is guaranteed to be a quiescent state. Because quiescent states are also grace periods on uniprocessor systems, as noted by Josh Triplett, any call to `synchronize_rcu()` is automatically a grace period. 

**Quick Quiz 3** : If all calls to `synchronize_rcu()` are automatically grace periods, why isn't `synchronize_rcu()` simply an empty function? 

**Quick Quiz 4** : If all calls to `synchronize_rcu()` are automatically grace periods, why doesn't `synchronize_rcu()` process any pending RCU callbacks? 

The following shows the code for the `call_rcu()` group of functions: 

```
1 static void __call_rcu(struct rcu_head *head,
      2            void (*func)(struct rcu_head *rcu),
      3            struct rcu_ctrlblk *rcp)
      4 {
      5   unsigned long flags;
      6 
      7   head->func = func;
      8   head->next = NULL;
      9   local_irq_save(flags);
     10   *rcp->curtail = head;
     11   rcp->curtail = &head->next;
     12   local_irq_restore(flags);
     13 }
     14 
     15 void call_rcu(struct rcu_head *head,
     16         void (*func)(struct rcu_head *rcu))
     17 {
     18   __call_rcu(head, func, &rcu_ctrlblk);
     19 }
     20 
     21 void call_rcu_bh(struct rcu_head *head,
     22      void (*func)(struct rcu_head *rcu))
     23 {
     24   __call_rcu(head, func, &rcu_bh_ctrlblk);
     25 }
```

Lines 1-13 show the code for `__call_rcu()`, which is a common-code helping function. Lines 7-8 initialize the specified `rcu_head` structure. Line 9 disables interrupts (and line 12 restores them), so that lines 10-11 can enqueue the callback undisturbed by interrupt handlers that might also invoke `call_rcu()`. 

Lines 15-19 and lines 21-25 are simple wrappers implementing `call_rcu()` (which enqueues the callback to `rcu_ctrlblk`) and `call_rcu_bh()` (which enqueues the callback to `rcu_bh_ctrlblk`), respectively. The callbacks are enqueued to the last segment of the queue, in other words, to the portion still waiting for a grace period to end. 

#### Grace-Period Machinery

The lowest-level grace-period machinery is supplied by the `rcu_qsctr_inc()` family of interfaces that report passage through a quiescent state. These functions are implemented as follows: 

```
1 static int rcu_qsctr_help(struct rcu_ctrlblk *rcp)
      2 {
      3   if (rcp->rcucblist != NULL &&
      4       rcp->donetail != rcp->curtail) {
      5     rcp->donetail = rcp->curtail;
      6     return 1;
      7   }
      8   return 0;
      9 }
     10 
     11 void rcu_qsctr_inc(int cpu)
     12 {
     13   if (rcu_qsctr_help(&rcu_ctrlblk) ||
     14       rcu_qsctr_help(&rcu_bh_ctrlblk))
     15         raise_softirq(RCU_SOFTIRQ);
     16 }
     17 
     18 void rcu_bh_qsctr_inc(int cpu)
     19 {
     20   if (rcu_qsctr_help(&rcu_bh_ctrlblk))
     21         raise_softirq(RCU_SOFTIRQ);
     22 }
```

Lines 1-9 show `rcu_qsctr_help()`, which is a common-code helper function for `rcu_qsctr_inc()` (lines 11-16) and `rcu_bh_qsctr_inc()` (lines 18-22), which in turn report ``rcu'' and ``rcu_bh'' quiescent states, respectively. 

Within `rcu_qsctr_help`, lines 3-4 check to see if there are any RCU callbacks within the specified `rcu_ctrlblk` structure that are still waiting for their grace period to complete, and, if so, line 5 updates the pointers so as to mark them as being ready to invoke, and line 6 returns non-zero in order to tell the caller to cause the callback-processing code to start running. Otherwise, line 8 returns zero to tell the caller that it is not necessary to process callbacks. 

The `rcu_qsctr_inc()` function invokes `rcu_qsctr_help()` twice, once for rcu and once again for rcu_bh, and, if either indicates that callback processing is required, it also invokes `raise_softirq(RCU_SOFTIRQ)` to cause callback processing to be performed. 

**Quick Quiz 5** : Why not simply directly call `rcu_process_callbacks()`? 

The `rcu_bh_qsctr_inc()` function invokes `rcu_qsctr_help()`, but only for rcu_bh. If the return value indicates that callback processing is required, `raise_softirq(RCU_SOFTIRQ)` is invoked to make this happen. 

#### Dynticks Interface

The tinyrcu dynticks interface is refreshingly simple, because uniprocessor systems need not worry about the dynticks-idle state of the nonexistent other CPUs. The code is as follows: 

```
1 void rcu_enter_nohz(void)
      2 {
      3   if (--dynticks_nesting == 0)
      4     rcu_qsctr_inc(0);
      5 }
      6 
      7 void rcu_exit_nohz(void)
      8 {
      9   dynticks_nesting++;
     10 }
     11 
     12 void rcu_irq_enter(void)
     13 {
     14   rcu_exit_nohz();
     15 }
     16 
     17 void rcu_irq_exit(void)
     18 {
     19   rcu_enter_nohz();
     20 }
     21 
     22 void rcu_nmi_enter(void)
     23 {
     24 }
     25 
     26 void rcu_nmi_exit(void)
     27 {
     28 }
```

The `rcu_enter_nohz()` function is shown on lines 1-5. Line 3 decrements the `dynticks_nesting` global variable, and, if the result is zero, the CPU has entered dynticks-idle mode, which is a quiescent state. (The `dynticks_nesting` global variable is initialized to 1.) 

**Quick Quiz 6** : Why is only `rcu_qsctr_inc()` invoked? What about `rcu_bh_qsctr_inc()`? 

The `rcu_exit_nohz()` function is shown on lines 7-10. This function simply increments the `dynticks_nesting` global variable. 

As can be seen on lines 12-20, the `rcu_irq_enter()` and `rcu_irq_exit()` functions are simple wrappers around `rcu_exit_nohz()` and `rcu_enter_nohz()`, respectively. Please note that entering an interrupt handler exits dynticks-idle mode from an RCU viewpoint, and vice versa. This often-confusing relationship is maintained in the names of these functions. 

Lines 22-28 show `rcu_nmi_enter()` and `rcu_nmi_exit()`, both of which are empty functions. Because we are running on a uniprocessor, we can safely ignore NMI handlers. The reason this is safe is that there cannot be any quiescent states on this CPU within an NMI handler, and there are no other CPUs to execute concurrent quiescent states. 

Finally, the `rcu_needs_cpu` function (not shown) simply returns non-zero, indicating that RCU is always prepared for a given CPU to go into dynticks-idle mode. 

###  Testing

Running the [script](<http://lwn.net/Articles/305787/>) included in the [hierarchical RCU article](<http://lwn.net/Articles/305782/>) shows that the `CONFIG_NO_HZ` kernel config parameter is important, and further investigation identifies the `CONFIG_PREEMPT` parameter as well. 

This results in a nice small set of testing scenarios: 

  1. `!CONFIG_PREEMPT` and `!CONFIG_NO_HZ`
  2. `!CONFIG_PREEMPT` and `CONFIG_NO_HZ`
  3. `CONFIG_PREEMPT` and `!CONFIG_NO_HZ`
  4. `CONFIG_PREEMPT` and `CONFIG_NO_HZ` 

This is a welcome contrast to the situation for hierarchical RCU. 

###  Memory Footprint

Simpler code means smaller memory footprint, as can be seen in the following table: 

> | Memory use|   
> ---|---|---  
> Implementation | text | data | bss| total | filename  
> Classic RCU | 363 | 12 | 24 | 399 | kernel/rcupdate.o  
> 1237 | 64 | 124 | 1425 | kernel/rcuclassic.o  
> |  |  | 1824 | Classic RCU Total  
>    
> Hierarchical RCU | 363 | 12 | 24 | 399 | kernel/rcupdate.o  
> 2344 | 240 | 184 | 2768 | kernel/rcutree.o  
> |  |  | 3167 | Hierarchical RCU Total  
>    
> Tiny RCU | 294 | 12 | 24 | 330 | kernel/rcupdate.o  
> 563 | 36 | 0 | 599 | kernel/rcutiny.o  
> |  |  | 929 | Tiny RCU Total  
  
Tiny RCU is about twice as small as Classic RCU (almost 900 bytes saved), and more than three times smaller than Hierarchical RCU (more than 2KB saved). 

###  Conclusion

As RCU's capabilities have grown, so has its code size. This Bloatwatch edition of RCU forces this trend in reverse, producing a very small implementation via the `!CONFIG_SMP` restriction. This implementation also provides a minimal Linux-kernel-capable implementation that may provide a good starting point for people wishing to learn about RCU implementations. 

### Acknowledgements

I am grateful to Josh Triplett for the conversation that got this project started, to Ingo Molnar and David Howells for their encouragement to see this through, to Will Schmidt for his help getting this to human-readable state, and to Kathy Bennett for her support of this effort. 

This work represents the view of the authors and does not necessarily represent the view of IBM. 

###  Answers to Quick Quizzes

**Quick Quiz 1** : But what if I need my RCU read-side critical section to sleep and block? 

**Answer** : A special form of RCU called "[SRCU](<http://lwn.net/Articles/202847/>)" does permit general sleeping in SRCU read-side critical sections. However, it is usually better to rework your RCU read-side critical section to avoid sleeping and blocking. One other exception is [preemptable RCU](<http://lwn.net/Articles/253651/>), which allows RCU read-side critical sections to be preempted and to block while acquiring what in non-RT kernels would be spinlocks. 

**Back to Quick Quiz 1**.

**Quick Quiz 2** : But we should be able to simply execute _all_ callbacks at each quiescent state! Why to we need to separate `->donetail` and `->curtail` sublists? 

**Answer** : Recall that the callbacks are not invoked directly from the scheduler, but rather from softirq context. It would be illegal to invoke callbacks that were registered after the quiescent state but before softirq commenced execution. Such callbacks could be registered from within irq handlers by invoking `call_rcu()`, and these irq handlers could be invoked between the time that the quiescent state occurred and the time that the softirq handler started executing. 

**Back to Quick Quiz 2**.

**Quick Quiz 3** : If all calls to `synchronize_rcu()` are automatically grace periods, why isn't `synchronize_rcu()` simply an empty function? 

**Answer** : If `synchronize_rcu()` were an empty function, then rcutorture would incorrectly conclude that RCU was not working, as it would appear to rcutorture that grace periods were never ending. Furthermore, `synchronize_rcu()` must at least prevent the compiler from reordering code across the call to `synchronize_rcu()`. 

**Back to Quick Quiz 3**.

**Quick Quiz 4** : If all calls to `synchronize_rcu()` are automatically grace periods, why doesn't `synchronize_rcu()` process any pending RCU callbacks? 

**Answer** : Because there currently appears to be no need to do so, given that callbacks will be processed on the next context switch or the next scheduling-clock interrupt from either user-mode execution or from the idle loop. Should experience demonstrate that these are insufficient, then one approach would be to add a `raise_softirq(RCU_SOFTIRQ)` statement to `synchronize_rcu()`. 

**Back to Quick Quiz 4**.

**Quick Quiz 5** : Why not simply directly call `rcu_process_callbacks()`? 

**Answer** : Deferring execution to the softirq environment disentangles RCU callback invocation from the scheduler, permitting the callbacks to freely invoke things like `wake_up()`. In addition, for rcu_bh, quiescent states might occur with locks held, and the RCU callbacks might well need to acquire these locks, potentially resulting in deadlock. In both cases, deferring to the softirq environment ensures a clean state for the callback. 

**Back to Quick Quiz 5**.

**Quick Quiz 6** : Why is only `rcu_qsctr_inc()` invoked? What about `rcu_bh_qsctr_inc()`? 

**Answer** : Because `rcu_qsctr_inc()` includes an implicit `rcu_bh_qsctr_inc()`, as can be seen from the code in the previous section. 

**Back to Quick Quiz 6**.
