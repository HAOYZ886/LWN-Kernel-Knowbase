---
title: Calibrating your fear of big bad optimizing compilers
url: https://lwn.net/Articles/799218/
date: "October 11, 2019"
category: "Development tools-Linux kernel memory model"
author: "October 11, 2019 (Many contributors)"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

October 11, 2019

(Many contributors)

This article was contributed by Jade Alglave, Will Deacon, Boqun Feng, David Howells, Daniel Lustig, Luc Maranget, Paul E. McKenney, Andrea Parri, Nicholas Piggin, Alan Stern, Akira Yokosawa, and Peter Zijlstra. 

As noted [earlier](<https://lwn.net/Articles/793253/>), when compiling Linux-kernel code that does a plain C-language load or store, as in "`a=b`", the C standard grants the compiler the right to assume that the affected variables are neither accessed nor modified by any other thread at the time of that load or store. The compiler is therefore permitted to carry out a surprisingly large number of optimizations, any number of which might ruin your concurrent code's day. Given that current compilers usually do not emit diagnostics warning of potential ruined days, it would be good to have other tools take on this task. One such tool is the [Kernel Thread Sanitizer (KTSAN)](<https://github.com/google/ktsan/wiki>), but its great strength, the ability to analyze huge bodies of code such as the Linux kernel, is also its great weakness, namely the need to use approximate (though still quite good) analysis techniques. 

**Quick Quiz 1** : But we shouldn't be afraid at all for things like on-stack or per-CPU variables, right?   
Answer

What is needed is a tool that can do exact analyses of huge bodies of code, but unfortunately the universe is under no compunction to give us what we think we need. We have therefore upgraded the [Linux Kernel Memory Model (LKMM)](<https://github.com/torvalds/linux/tree/master/tools/memory-model>) to do exact analyses of small bodies of code, and this upgrade was accepted into the Linux kernel during the v5.3 merge window. The challenge of doing exact analyses of large bodies of code thus remains open, but in the meantime we have another useful tool at our disposal. 

The following sections describe how to use this upgrade to LKMM: 

  1. Goals and non-goals
  2. A plain example
  3. A less-plain example
  4. Locking
  5. Reference counting
  6. Read-copy update (RCU)
  7. Debug output
  8. Access-marking policies
  9. Limitations
  10. Summary 

This is followed by the inevitable answers to the quick quizzes. 

#### Goals and non-goals

TL;DR: The goal of LKMM is to help people understand Linux-kernel concurrency. 

A key point from the first article is that we simply do not have a full catalog of all compiler optimizations that currently exist, let alone of all possible compiler options that might one day exist. Furthermore, the kernel must, in many cases, [live outside of the bounds of the C standard](<http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0124r6.html>), and thus cannot simply make direct use of the C11 memory model. The details of the compiler, of the compiler options used in Linux-kernel builds, and all of the architecture-specific code therefore impinge on LKMM--and vice versa. 

Because of all of these complications and uncertainties, LKMM cannot possibly be a cut-and-dried judge and juror for all Linux-kernel memory-ordering matters. It should instead be seen as an advisor. Now, in generic, non-performance-critical code, it might be wise to pay extremely close attention to LKMM's advice. On the other hand, developers writing fastpath code might need to take an LKMM warning as a sign of the need to be careful, rather than as an error message to be fixed at all costs. 

With that in mind, let's look at some LKMM litmus-test examples involving plain C-language accesses. 

#### A plain example

Consider the following message-passing litmus test using plain C-language accesses and read and write memory barriers: 

> Litmus Test #1 

```
>       1 C C-MP+p-wmb-p+p-rmb-p
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x0, int *x1)
>       7 {
>       8         *x0 = 1;
>       9         smp_wmb();
>      10         *x1 = 1;
>      11 }
>      12
>      13 P1(int *x0, int *x1)
>      14 {
>      15         int r1;
>      16         int r2;
>      17
>      18         r1 = *x1;
>      19         smp_rmb();
>      20         r2 = *x0;
>      21 }
>      22
>      23 exists (1:r1=1 /\ 1:r2=0)
>
```

As always, this litmus test can be run from within the kernel's `tools/memory-model` directory using the following command: 

> 

```
>     herd7 -conf linux-kernel.cfg /path/to/litmus/tests/C-MP+p-wmb-p+p-rmb-p.litmus
>
```

Here, the "`/path/to/litmus/tests`" is of course replaced by the path to the directory containing your litmus tests. (See the `README` file in that same directory for installation instructions and [an earlier LWN article](<https://lwn.net/Articles/720550/>) for more details on litmus tests.) The output of this command is shown below: 

> Outcome for Litmus Test #1 (linux-kernel model) 

```
>      1 Test C-MP+p-wmb-p+p-rmb-p Allowed
>      2 States 4
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=0; 1:r2=1;
>      5 1:r1=1; 1:r2=0;
>      6 1:r1=1; 1:r2=1;
>      7 Ok
>      8 Witnesses
>      9 Positive: 1 Negative: 3
>     10 Flag data-race
>     11 Condition exists (1:r1=1 /\ 1:r2=0)
>     12 Observation C-MP+p-wmb-p+p-rmb-p Sometimes 1 3
>     13 Time C-MP+p-wmb-p+p-rmb-p 0.00
>     14 Hash=055863a755bfaf3667f1667e6d660349
>
```

**Quick Quiz 2** : But suppose only one of the accesses to a given variable is a plain C-language access, and that access is the only store. Should this be considered a data race?   
Answer

**Quick Quiz 3** : Why the cop-out? Why not just do the work required to ensure that the list of states and the `Observation` line are all accurate?   
Answer

Line 10 contains the key advice: "`Flag data-race`". This advice means that the `Observation` line's verdict is untrustworthy and that the list of states on lines 3-6 is unreliable. The problem is that this litmus test has at least one data race, meaning that there are multiple concurrent accesses to a given variable, at least one of which is a plain C-language access and at least one of which is a store. 

The variable `x1` is clearly subject to a data race because it can be accessed concurrently by a pair of plain accesses, at least one of which is a write. However, `x1` only ever takes on the values zero and one, so the data race on that variable [might be tolerable](<http://lkml.kernel.org/r/20190606061438.nyzaeppdbqjt3jbp@gondor.apana.org.au>), at least assuming a healthy fear of the big bad optimizing compiler. But what if we want to check for other data races? 

The solution is to tell LKMM that you are excluding `x1` from data-race flagging. One way to do this is to add `READ_ONCE()` and `WRITE_ONCE()` to the litmus test (as opposed to your Linux-kernel code), preferably with a comment explaining the situation: 

> Litmus Test #2 

```
>       1 C C-MP+p-wmb-o+o-rmb-p
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x0, int *x1)
>       7 {
>       8         *x0 = 1;
>       9         smp_wmb();
>      10         WRITE_ONCE(*x1, 1); // Tolerate data race
>      11 }
>      12
>      13 P1(int *x0, int *x1)
>      14 {
>      15         int r1;
>      16         int r2;
>      17
>      18         r1 = READ_ONCE(*x1); // Tolerate data race
>      19         smp_rmb();
>      20         r2 = *x0;
>      21 }
>      22
>      23 exists (1:r1=1 /\ 1:r2=0)
>
```

LKMM still flags a data race: 

> Outcome for Litmus Test #2 (linux-kernel model) 

```
>      1 Test C-MP+p-wmb-o+o-rmb-p Allowed
>      2 States 3
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=0; 1:r2=1;
>      5 1:r1=1; 1:r2=1;
>      6 No
>      7 Witnesses
>      8 Positive: 0 Negative: 3
>      9 Flag data-race
>     10 Condition exists (1:r1=1 /\ 1:r2=0)
>     11 Observation C-MP+p-wmb-o+o-rmb-p Never 0 3
>     12 Time C-MP+p-wmb-o+o-rmb-p 0.00
>     13 Hash=743f0171133035c53a5a29972b0ba0fd
>
```

The reason for this is that the plain C-language accesses to `x0` can also execute concurrently, and one of these accesses is a write. We could fix this by also marking the `x0` accesses with `READ_ONCE()` and `WRITE_ONCE()`, but an alternative is to avoid concurrency on `x0` as follows: 

> Litmus Test #3 

```
>       1 C C-MP+p-wmb-o+o+ctrl-rmb-p
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x0, int *x1)
>       7 {
>       8         *x0 = 1;
>       9         smp_wmb();
>      10         WRITE_ONCE(*x1, 1); // Tolerate data race
>      11 }
>      12
>      13 P1(int *x0, int *x1)
>      14 {
>      15         int r1;
>      16         int r2;
>      17
>      18         r1 = READ_ONCE(*x1); // Tolerate data race
>      19         if (r1) {
>      20                 smp_rmb();
>      21                 r2 = *x0;
>      22         }
>      23 }
>      24
>      25 exists (1:r1=1 /\ 1:r2=0)
>
```

The `if` statement on line 19, when combined with the `smp_wmb()` on line 9, is intended to guarantee that lines 8 and 21 never execute concurrently. Running the model yields: 

> Outcome for Litmus Test #3 (linux-kernel model) 

```
>      1 Test C-MP+p-wmb-o+o+ctrl-rmb-p Allowed
>      2 States 2
>      3 1:r1=0; 1:r2=0;
>      4 1:r1=1; 1:r2=1;
>      5 No
>      6 Witnesses
>      7 Positive: 0 Negative: 2
>      8 Condition exists (1:r1=1 /\ 1:r2=0)
>      9 Observation C-MP+p-wmb-o+o+ctrl-rmb-p Never 0 2
>     10 Time C-MP+p-wmb-o+o+ctrl-rmb-p 0.00
>     11 Hash=01fe003cd2759d9284d40c081007c282
>
```

**Quick Quiz 4** : But the outcome "`1:r1=0; 1:r2=1;`" also disappeared. Why?   
Answer

There is no longer a data race flagged, and the cyclic outcome no longer occurs. Therefore, if we are willing to live in sufficient fear of the compiler to keep accesses to `x1` sane on the one hand and if we add a conditional check protecting `x0` on the other, we can obtain the required outcome. 

In this case, because of the `smp_wmb()` and the fact that there was only a single use of the value read, all that was needed to tell LKMM about a tolerated data race was to use `WRITE_ONCE()` and `READ_ONCE()` in the corresponding litmus test. Unfortunately, some situations require a bit more work, as will be hinted at in the next section. 

#### A less-plain example

The compiler has great freedom to optimize plain accesses, and with this great freedom comes great complexity. To see this, consider the following litmus test: 

> Litmus Test #4 

```
>       1 C C-read-multiuse
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *a)
>       7 {
>       8         *a = 1;
>       9 }
>      10
>      11 P1(int *a, int *b, int *c)
>      12 {
>      13         int r1;
>      14
>      15         r1 = *a;
>      16         *b = r1;
>      17         *c = r1;
>      18 }
>      19
>      20 locations [1:r1; a; b; c]
>      21 exists(b=1 /\ c=0)
>
```

As we should by now expect, this results in a data race: 

> Outcome for Litmus Test #4 (linux-kernel model) 

```
>      1 Test C-read-multiuse Allowed
>      2 States 2
>      3 1:r1=0; a=1; b=0; c=0;
>      4 1:r1=1; a=1; b=1; c=1;
>      5 No
>      6 Witnesses
>      7 Positive: 0 Negative: 2
>      8 Flag data-race
>      9 Condition exists (b=1 /\ c=0)
>     10 Observation C-read-multiuse Never 0 2
>     11 Time C-read-multiuse 0.00
>     12 Hash=0cab074d9a510f141aae9026ce447828
>
```

Creating a data-race-tolerant litmus test is straightforward: 

> Litmus Test #5 

```
>       1 C C-read-multiuse-drt1
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *a)
>       7 {
>       8         WRITE_ONCE(*a, 1); // Tolerate data race
>       9 }
>      10
>      11 P1(int *a, int *b, int *c)
>      12 {
>      13         int r1;
>      14
>      15         r1 = READ_ONCE(*a); // Tolerate data race
>      16         *b = r1;
>      17         *c = r1;
>      18 }
>      19
>      20 locations [1:r1; a; b; c]
>      21 exists(b=1 /\ c=0)
>
```

And this both tolerates the data race and excludes the undesirable outcome: 

> Outcome for Litmus Test #5 (linux-kernel model) 

```
>      1 Test C-read-multiuse-drt1 Allowed
>      2 States 2
>      3 1:r1=0; a=1; b=0; c=0;
>      4 1:r1=1; a=1; b=1; c=1;
>      5 No
>      6 Witnesses
>      7 Positive: 0 Negative: 2
>      8 Condition exists (b=1 /\ c=0)
>      9 Observation C-read-multiuse-drt1 Never 0 2
>     10 Time C-read-multiuse-drt1 0.00
>     11 Hash=96b3ae01a3c486885df1aec4d978bad9
>
```

Job done, right? 

Not quite, courtesy of the aforementioned complexity. Recall that compilers can [invent loads](<https://lwn.net/Articles/793253/#Invented%20Loads>). Therefore, a better translation to a data-race-tolerant litmus test would duplicate the load from shared variable `a` as follows: 

> Litmus Test #6 

```
>       1 C C-read-multiuse-drt2
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *a)
>       7 {
>       8         WRITE_ONCE(*a, 1); // Tolerate data race
>       9 }
>      10
>      11 P1(int *a, int *b, int *c)
>      12 {
>      13         int r1;
>      14         int r2;
>      15
>      16         r1 = READ_ONCE(*a); // Tolerate data race
>      17         *b = r1;
>      18         r2 = READ_ONCE(*a); // Tolerate data race
>      19         *c = r2;
>      20 }
>      21
>      22 locations [1:r1; a; b; c]
>      23 exists(b=1 /\ c=0)
>
```

And this still tolerates the data race and excludes the undesirable outcome: 

> Outcome for Litmus Test #6 (linux-kernel model) 

```
>      1 Test C-read-multiuse-drt2 Allowed
>      2 States 3
>      3 1:r1=0; a=1; b=0; c=0;
>      4 1:r1=0; a=1; b=0; c=1;
>      5 1:r1=1; a=1; b=1; c=1;
>      6 No
>      7 Witnesses
>      8 Positive: 0 Negative: 3
>      9 Condition exists (b=1 /\ c=0)
>     10 Observation C-read-multiuse-drt2 Never 0 3
>     11 Time C-read-multiuse-drt2 0.01
>     12 Hash=17ff8b2e2c285776994d4488fcdcd3bb
>
```

So _now_ the job is done, right? 

Still not quite. Recall that compilers can [reorder code](<https://lwn.net/Articles/793253/#Code%20Reordering>). And there is nothing telling the compiler that the store to `b` needs to precede the store to `c`; additionally, the fact that the actual code uses a plain C-language load from shared variable `a` allows the compiler to assume (incorrectly, in this case) that shared variable `a` isn't changing. We therefore need to account for that with another data-race-tolerant litmus test that does the reordering, for example, as follows: 

> Litmus Test #7 

```
>       1 C C-read-multiuse-drt3
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *a)
>       7 {
>       8         WRITE_ONCE(*a, 1); // Tolerate data race
>       9 }
>      10
>      11 P1(int *a, int *b, int *c)
>      12 {
>      13         int r1;
>      14         int r2;
>      15
>      16         r2 = READ_ONCE(*a); // Tolerate data race
>      17         *c = r2;
>      18         r1 = READ_ONCE(*a); // Tolerate data race
>      19         *b = r1;
>      20 }
>      21
>      22 locations [1:r1; a; b; c]
>      23 exists(b=1 /\ c=0)
>
```

This still tolerates the data race, but allows the undesirable outcome: 

> Outcome for Litmus Test #7 (linux-kernel model) 

```
>      1 Test C-read-multiuse-drt3 Allowed
>      2 States 3
>      3 1:r1=0; a=1; b=0; c=0;
>      4 1:r1=1; a=1; b=1; c=0;
>      5 1:r1=1; a=1; b=1; c=1;
>      6 Ok
>      7 Witnesses
>      8 Positive: 1 Negative: 2
>      9 Condition exists (b=1 /\ c=0)
>     10 Observation C-read-multiuse-drt3 Sometimes 1 2
>     11 Time C-read-multiuse-drt3 0.01
>     12 Hash=61f32f3a79e57808d348f31f5800ae1d
>
```

**Quick Quiz 5** : But wait! Given that `READ_ONCE()` provides no ordering, why would Litmus Test #5 avoid the undesirable outcome?   
Answer

This example illustrates the need to carefully think through the possible compiler optimizations when using plain C-language loads and stores in situations involving data races. It also illustrates the possible need to use multiple litmus tests to fully analyze the possible outcomes. 

So should use of plain C-language loads and stores for shared variables be completely abolished throughout the kernel? 

Absolutely not. 

For one thing, it is often the case that a given plain C-language load and store will be data-race free, as illustrated by the next few sections. 

#### Locking

The prevalence and usefulness of locking, particularly on systems with modest numbers of CPUs, is one reason why C and C++ did not add concurrency features for the first few decades of their existence. We should therefore expect that fully locked concurrent code should do fine with plain C-language accesses. For example, consider this fully locked store-buffering litmus test: 

> Litmus Test #8 

```
>       1 C C-SB+l-p-p-u+l-p-p-u
>       2
>       3 {
>       4 }
>       5
>       6 P0(int *x0, int *x1, spinlock_t *s)
>       7 {
>       8         int r1;
>       9
>      10         spin_lock(s);
>      11         *x0 = 1;
>      12         r1 = *x1;
>      13         spin_unlock(s);
>      14 }
>      15
>      16 P1(int *x0, int *x1, spinlock_t *s)
>      17 {
>      18         int r1;
>      19
>      20         spin_lock(s);
>      21         *x1 = 1;
>      22         r1 = *x0;
>      23         spin_unlock(s);
>      24 }
>      25
>      26 exists (0:r1=0 /\ 1:r1=0)
>
```

As expected, LKMM shows that this litmus test produces only the expected serialized outcomes, and does so without data races: 

> Outcome for Litmus Test #8 (linux-kernel model) 

```
>      1 Test C-SB+l-p-p-u+l-p-p-u Allowed
>      2 States 2
>      3 0:r1=0; 1:r1=1;
>      4 0:r1=1; 1:r1=0;
>      5 No
>      6 Witnesses
>      7 Positive: 0 Negative: 2
>      8 Condition exists (0:r1=0 /\ 1:r1=0)
>      9 Observation C-SB+l-p-p-u+l-p-p-u Never 0 2
>     10 Time C-SB+l-p-p-u+l-p-p-u 0.01
>     11 Hash=a1b190dd8375d869bc8826836e05f943
>
```

But locking is not the only data-race-free synchronization primitive. 

#### Reference counting

Reference counting has, if anything, been in use longer than has locking. It should therefore be no surprise that it is possible to atomically manipulate reference counts in such a way as to permit plain C-language access to shared variables. One approach uses `atomic_dec_and_test()` so that the task that decrements the reference count to zero owns all the data. The following (fanciful) litmus test illustrates this design pattern: 

> Litmus Test #9 

```
>       1 C C-SB+p-rc-p-p+p-rc-p-p
>       2
>       3 {
>       4         atomic_t rc=2;
>       5 }
>       6
>       7 P0(int *x0, int *x1, atomic_t *rc)
>       8 {
>       9         int r0;
>      10         int r1;
>      11
>      12         *x0 = 1;
>      13         if (atomic_dec_and_test(rc)) {
>      14                 r0 = *x0;
>      15                 r1 = *x1;
>      16         }
>      17 }
>      18
>      19 P1(int *x0, int *x1, atomic_t *rc)
>      20 {
>      21         int r0;
>      22         int r1;
>      23
>      24         *x1 = 1;
>      25         if (atomic_dec_and_test(rc)) {
>      26                 r0 = *x0;
>      27                 r1 = *x1;
>      28         }
>      29 }
>      30
>      31 exists ~((0:r0=1 /\ 0:r1=1 /\ 1:r0=0 /\ 1:r1=0) \/
>      32          (0:r0=0 /\ 0:r1=0 /\ 1:r0=1 /\ 1:r1=1))
>
```

Initially, each process owns its variable, that is, `P0()` owns `x0` and `P1()` owns `x1`. This reference count `rc` is initialized to the value 2 on line 4, indicating that both processes still own their respective variables. Each process updates its variable (lines 12 and 24), then releases its reference on `rc` (lines 13 and 25). The "winning" process that decrements `rc` to zero reads out both values, so that its values of `r0` and `r1` are equal to the value 1. The "losing" process's local variables remain zero. The `exists` clause on line 31 verifies this relationship: 

> Outcome for Litmus Test #9 (linux-kernel model) 

```
>      1 Test C-SB+p-rc-p-p+p-rc-p-p Allowed
>      2 States 2
>      3 0:r0=0; 0:r1=0; 1:r0=1; 1:r1=1;
>      4 0:r0=1; 0:r1=1; 1:r0=0; 1:r1=0;
>      5 No
>      6 Witnesses
>      7 Positive: 0 Negative: 2
>      8 Condition exists (not (0:r0=1 /\ 0:r1=1 /\ 1:r0=0 /\ 1:r1=0 \/ 0:r0=0 /\ 0:r1=0 /\ 1:r0=1 /\ 1:r1=1))
>      9 Observation C-SB+p-rc-p-p+p-rc-p-p Never 0 2
>     10 Time C-SB+p-rc-p-p+p-rc-p-p 0.02
>     11 Hash=7692409758270a77b577b11ab7cca3e3
>
```

**Quick Quiz 6** : I have seen plain C-language incrementing and decrementing of reference counters. How can that possibly work?   
Answer

As expected, there are no data races and the "winner" always safely reads out the data.
